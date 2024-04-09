---
title: Talent Plan（ 路径二）Project 1
author: siegelion
date: 2022/01/30 20:15
tags: [比赛,Go]
categories: [笔记]
---



Project1的主要内容是实现一个单机的K/V存储引擎，并通过[gRPC](https://grpc.io/docs/guides/)为上层提供服务，通过RPC调用该存储引擎可以执行Put、Delete、Get、Scan四种操作。存储引擎的底层借助[badger](https://github.com/dgraph-io/badger)作为底层K/V数据库实现。

该项目的任务可划分为以下两个步骤，包括：

1. 实现一个单机的存储引擎。
2. 借助实现的存储引擎，实现K/V服务。

虽然在Project1阶段，涉及的代码量并不多，只要明确工作内容，然后在实现时注意细节便可通过全部测试点。但是Project1阶段实现的单机存储引擎与Project4实现的分布式存储引擎的数据结构有异曲同工之妙，所以对在完成Project1的同时，对`Storage`与`StorageReader`的设计进行理解对后续任务的完成有一些帮助。

## K/V存储引擎

TinyKV中对存储引擎进行了抽象，抽象为以下的接口：

> 定义在 kv/storage/storage.go

```go
type Storage interface {
	Start() error
	Stop() error
	Write(ctx *kvrpcpb.Context, batch []Modify) error
	Reader(ctx *kvrpcpb.Context) (StorageReader, error)
}
```

本Project中需要实现的`StandAloneStorage`就是实现了以上接口的一个存储引擎。我们对`StandAloneStorage`的数据结构进行如下设计：

> 实现在 kv/storage/standalone_storage/standalone_storage.go

```go
type StandAloneStorage struct {
	conf *config.Config  // 配置文件
	KvDB *badger.DB     // 底层数据库
}
```

`conf`字段保存存储引擎相关的配置信息对象的指针，`KvDB`字段保存底层数据库对象的指针。

与`StandAloneStorage`相关的函数主要有以下几个，简要对其进行说明：

- `NewStandAloneStorage`

    初始化存储引擎，函数执行时会创建`StandAloneStorage`对象，初始化`conf`字段，将`KvDB`字段暂时设置为`nil`。

- `Start`

    启动存储引擎，根据`conf`字段中保存的`dbPath`信息，设置`badger`相关的配置信息，根据设置好的配置新建badger数据库，并将数据保存在目标文件夹中。并将`StandAloneStorage`对象的`KvDB`字段设置为新建的数据库对象的指针。

- `Stop`

    关闭底层数据库，根据`StandAloneStorage`对象`conf`字段中的信息，清空保存数据库的文件夹。

- `Reader`

    该函数的名字并非为`Read`而为`Reader`，是因为该函数的作用并不是读取数据，而是向上层返回一个可用来读取数据的`StorageReader`对象，通过该reader上层在进行调用时可以根据需要实现想要的Get或Scan操作。

    `StorageReader`实际上也实现了一个定义好的接口：

    > 定义在 kv/storage/storage.go

    ```go
    type StorageReader interface {
    	// When the key doesn't exist, return nil for the value
    	GetCF(cf string, key []byte) ([]byte, error)
    	IterCF(cf string) engine_util.DBIterator
    	Close()
    }
    ```

    为了实现`StandAloneStorage`的`Reader`函数，需要一个结构体实现了`StorageReader`接口，这个数据结构为`StandaloneStorageReader`：

    > StorageReader 定义在 /kv/storage/storage.go

    ```go
    type StorageReader interface {
    	// When the key doesn't exist, return nil for the value
    	GetCF(cf string, key []byte) ([]byte, error)
    	IterCF(cf string) engine_util.DBIterator
    	Close()
    }
    ```

    > `storage.StorageReader`定义在 kv/storage/standalone_storage/standalone_storage_reader.go

    ```go
    type StandaloneStorageReader struct {
    	Txn *badger.Txn
    }
    ```

    该结构体实际上是对`*badger.Txn`的一个封装，实现接口的函数时实际上也是在通过`*badger.Txn`来调用badger引擎的方法来实现数据存取。

    需要注意的是：

    - GetCF方法，在调用`engine_util.GetCFFromTxn`方法时，可能返回` badger.ErrKeyNotFound`的错误，若返回该错误，需要进行特判，将返回的val和err都设为nil。
    - IterCF方法，在调用`engine_util.NewCFIterator`初始化游标后，需要调用`Rewind()`将游标重置在初始位置。
    - Close方法，需要调用`Discard()`方法将连接关闭。

- `Write`

    该方法接收的参数中，存在一个`batch`字段，对应一个`storage.Modify`类型的切片，该切片中保存多种修改，共有两种写操作类型：Put和Delete，需要根据每个修改的类型进行不同的操作，调用`engine_util`的相应方法，来实现数据写入。

## K/V服务

即为单机存储引擎对外暴露的gRPC接口，来调用存储引擎进行Get、Put、Delete、Scan四种操作。

### Get

1. 调用`server.storage.Reader`方法初始化一个`StorageReader`对象。
2. 调用该`StorageReader`对象的GetCF方法，返回`val`和`err`值。
3. 根据`va`l和`err`的取值情况，来进行返回对象的设置。
    - 如上文所说，在Get操作时，可能存在找不到该值的情况，这时返回的`val`和`err`都应该为`nil`。这时需要将返回对象的`NotFound`字段设置为`true`。
    - 当`err`值不为`nil`，即可认为出现错误，将返回对象的`Error`字段设置为`err`。
    - 其余情况为成功。

### Put

初始化`storage.Modify`对象，将`Data`字段设置为`storage.Put`对象，根据操作类型，设置`Key`、`Value`、`Cf`三个字段。然后调用`StandAloneStorage`的`Write`方法将数据写入即可。

### Delete

与Put操作类似，只是将`Data`字段设置为`storage.Delete`对象。

### Scan

Scan操作同样需要用到我们上文提到的`StorageReader`对象。

1. 调用`server.storage.Reader`方法初始化一个`StorageReader`对象。
2. 调用`IterCF`方法返回一个游标。
3. 使用迭代该游标，每次从游标中取出`key`与`val`。
    - 由于Scan操作的请求时会携带一个`limit`字段，所以在迭代时我们需要对迭代次数进行判断。
    - 每次还需要对游标的有效性进行判断。

