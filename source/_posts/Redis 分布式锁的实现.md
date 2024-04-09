---
title: Redis 分布式锁的实现
author: siegelion
date: 2022/02/10 10:15
tags: [数据库,Redis]
categories: [笔记]
---

## 前言

> 本文是阅读《Redis 实战》期间的学习笔记，书中的主要功能使用使用`Python`进行实现，本文使用`Golang`对功能进行复现。

Redis为开发者们提供了事务的功能，具体的实现为：以特殊命令`MULTI`开始，接着输入要执行的多条命令，最后输入特殊命令`EXEC`标志着事物的结束，同时还需要配合`WATCH`命令以保持数据的一致性，在`WATCH`期间，当客户端执行`EXEC`时，若`Redis`检测到，有其他客户端抢先对`WATCH`的数据进行了修改，则会通知当前客户端本次事务执行失败。所以说本质上，`Redis`事务的数据一致性是通过借助锁来实现的，只不过这个锁本质上是一把乐观锁。

> - 乐观锁：乐观锁假设数据一般情况不会造成冲突，所以在数据进行提交更新的时候，才会正式对数据的冲突与否进行检测，如果冲突，则返回给用户异常信息，让用户决定如何去做。
> - 悲观锁：悲观锁，具有强烈的独占和排他特性。它指的是对数据被外界(包括本系统当前的其他事务，以及来自外部系统的事务处理)修改持保守态度。因此，在整个数据处理过程中，将数据处于锁定状态。
>     - 共享锁：多个事务可以同时读取数据，但是不能修改数据。
>     - 排他锁：只有一个事务可以独占数据，其他事务需要阻塞等待。

这种乐观锁，由于在执行事务前并不会检测检测数据的一致性，当检测到数据的不一致时，本次事务执行失败，本次事务逻辑计算、网络传输时间的消耗也就失去了意义，因为还需要再一次对事务进行执行直到事务执行通过。当并发量上升时，无可避免地会发生多个事务同时修改同一数据的情况，这时会出现事务大量重复尝试的情况，导致平均每个事务的执行时间大幅度增加。

这种情况下只借助Redis中提供的Watch机制以达到乐观锁的功能，会对性能造成一定的负担，为了以更好的形式实现想要的功能，我们需要自己动手实现一个悲观锁，虽然多个事务同时读取同一数据时，也会造成多个事务反复尝试的情况，但是悲观锁的先获取锁的机制，节省了逻辑计算与网络传输的时间。

---

虽然在逻辑上，悲观锁在高并发情况下的性能理应更优，但是在我的2天的实际的测试中，由于测试方法的不恰当，导致最终的结果，并不如上文预料的那样，反而与之相反。

![img](https://dl4.weshineapp.com/gif/20210918/174d3c4fad3d8cf5605870a397636923.gif?f=micro_5b+D57Sv)

## 代码实现

Anyway！还是分析一下我们的代码实现吧。

### 业务场景

我们进行实验的业务场景为：“一个在线商店，可以供用户出售或者购买商品。”Redis中的数据结构如下：

- 用户信息：散列类型
    - key: `<users> : <userId>`
    - value: 
        - name:
        - funds:
- 用户背包：set类型
    - key: `inventory : <sellerId>`
    - value: `<itemId>`
- 市场：`zset`类型
    - key: `market:`
    - value: `<itemId> . <sellerId>`
    - 分值: `<价格>`

### Watch 方式实现的乐观锁

#### 使用`Watch`方式实现上架商品

```go
func SaleGoodWithWatch(item string, sellerId int, price float64) bool {
	ctx := context.TODO()
	inventory := fmt.Sprintf("inventory:%d", sellerId)
	marketItemID := fmt.Sprintf("%s.%d", item, sellerId)
	transactionFuc := func(t *redis.Tx) error { // 事务函数
		res, _ := myRedis.SIsMember(ctx, inventory, item).Result()
		if res { // 用户是否拥有物品
			pipe := myRedis.Pipeline()
			pipe.ZAdd(ctx, "market:", &redis.Z{Score: price, Member: marketItemID})
			pipe.SRem(ctx, inventory, item)
			_, err := pipe.Exec(ctx) // 执行事务
			return err
		}
		return nil
	}

	for {
		err := myRedis.Watch(ctx, transactionFuc, inventory) // Watch
		if err == nil {
			return true
		}
	}
}
```

####  使用`Watch`方式实现购买商品 

```go
func PurchaseGoodWithWatch(buyerId, sellerId int, itemId string) bool {
	ctx := context.TODO()
	buyer := fmt.Sprintf("user:%d", buyerId)
	inventory := fmt.Sprintf("inventory:%d", buyerId)
	seller := fmt.Sprintf("user:%d", sellerId)
	marketItem := fmt.Sprintf("%s.%d", itemId, sellerId)

	pipe := myRedis.Pipeline()
	funds, _ := myRedis.HGet(ctx, buyer, "Funds").Result() // 资金
	price, _ := myRedis.ZScore(ctx, "market:", marketItem).Result() // 商品价格
	pipe.Exec(ctx)

	f, _ := strconv.ParseInt(funds, 10, 64)
	transactionFuc := func(t *redis.Tx) error { // 事务函数
		pipe := myRedis.Pipeline()
		pipe.HIncrBy(ctx, seller, "Funds", int64(price))
		pipe.HIncrBy(ctx, buyer, "Funds", int64(-price))
		pipe.SAdd(ctx, inventory, itemId)
		pipe.ZRem(ctx, "market:", marketItem).Result()
		_, err := pipe.Exec(ctx)
		return err
	}

	if float64(f) >= price { // 是否有足够的资金
		for {
			err := myRedis.Watch(ctx, transactionFuc, "market:", buyer, seller) // Watch
			if err == nil {
				return true
			}
		}
	}
	return false
}
```

### 自己实现的悲观锁

`Redis`中拥有的`SetNX`命令，是`set if not exist`的简写，因此当某个键值对不存在时使用该命令会设置键值，指定键对应的键值对存在时，该命令时会返回错误。这可以看作一个原子命令，可以用该命令实现获取锁的功能。

键值对的键名为锁的名字，键值对的值则为一个不会重复的UUID，并为该键值对设置一个过期时间。设置成功则意味着获取到锁。释放锁的操作，则由删除该键值对的操作实现。

虽然如上文所说，虽然在我们的测试中，通过`SetNX`方式实现的悲观锁的性能不如`Watch`，但并不意味着我们实现锁的方式是错误的。在此记录我们对锁的实现：

#### 获取锁

```go
func acquireLock(lockName string, acquireTimeout int) string {
	end := time.Now().Add(time.Duration(acquireTimeout) * time.Second)
	identifier := uuid.New().String()
	lockName = fmt.Sprintf("lock:%s", lockName)
	for time.Now().Before(end) {
		res, _ := myRedis.SetNX(context.TODO(), lockName, identifier, time.Duration(1)*time.Second).Result()
		if res {
			return identifier
		} else {
			time.Sleep(time.Millisecond)
		}
	}
	return ""
}
```

#### 释放锁

```go
func releaseLock(lockName, identifier string) bool {
	ctx := context.TODO()
	lockName = fmt.Sprintf("lock:%s", lockName)
	transactionFuc := func(t *redis.Tx) error {
		res, _ := t.Get(ctx, lockName).Result()
		if res == identifier {
			_, err := myRedis.TxPipelined(ctx, func(pipe redis.Pipeliner) error {
				pipe.Del(ctx, lockName)
				return nil
			})
			return err
		}
		return nil
	}

	for {
		err := myRedis.Watch(ctx, transactionFuc, lockName) // Watch 锁
		if err == nil {
			return true
		}
	}
}
```

#### 使用`SetNX`方式实现上架商品

```go
func SaleGoodWithLock(item string, sellerId int, price float64) bool {
	ctx := context.TODO()
	inventory := fmt.Sprintf("inventory:%d", sellerId)
	marketItemID := fmt.Sprintf("%s.%d", item, sellerId)
	res, _ := myRedis.SIsMember(ctx, inventory, item).Result()
	if res { // 用户是否拥有物品
		identifier := myRedis.acquireLock(inventory, 5) // 获取悲观锁
		if identifier == "" {                           // 是否获取到锁
			return false
		}
		pipe := myRedis.Pipeline()
		pipe.ZAdd(ctx, "market:", &redis.Z{Score: price, Member: marketItemID}).Result()
		pipe.SRem(ctx, inventory, item)
		pipe.Exec(ctx)

		myRedis.releaseLock(inventory, identifier) // 释放锁
		return true
	} else {
		return false
	}
}
```

#### 使用`SetNX`方式实现购买商品

```go
func PurchaseGoodWithLock(buyerId, sellerId int, itemId string) bool {
	ctx := context.TODO()
	buyer := fmt.Sprintf("user:%d", buyerId)
	inventory := fmt.Sprintf("inventory:%d", buyerId)
	seller := fmt.Sprintf("user:%d", sellerId)
	marketItem := fmt.Sprintf("%s.%d", itemId, sellerId)

	funds, _ := myRedis.HGet(ctx, buyer, "Funds").Result()
	price, _ := myRedis.ZScore(ctx, "market:", marketItem).Result()
	f, _ := strconv.ParseInt(funds, 10, 64)

	if float64(f) >= price { // 是否拥有足够的资金
		identifier := myRedis.acquireLock("market:", 5) // 获取锁
		if identifier == "" {                           // 是否获取到锁
			return false
		}
		pipe := myRedis.Pipeline()
		pipe.HIncrBy(ctx, seller, "Funds", int64(price))
		pipe.HIncrBy(ctx, buyer, "Funds", int64(-price))
		pipe.SAdd(ctx, inventory, itemId)
		pipe.ZRem(ctx, "market:", marketItem)
		pipe.Exec(ctx)
		myRedis.releaseLock("market:", identifier) // 释放锁
		return true
	}
	return false
}
```

