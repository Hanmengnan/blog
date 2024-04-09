---
title:  Vue中的数据绑定
tags:  [前端,Vue]
categories: [笔记]
date: 2020/11/15 20:46:25
---

在Vue中数据绑定分为两种：“单向数据绑定”和“双向数据绑定”。



对于这二者的区别，我有以下的思考：

- 单向数据绑定：适用于父组件触发事件，改变自身`property`的值，进而将改变后的值通过`prop`的方式传给子组件，进而使子组件的`property`改变。
- 双向数据绑定：子组件触发事件，改变自身的`property`的值，需要用一种信号的方式，将触发事件与值发送出去，父组件监听这个事件并接收值，并用值改变自己的`property`。

## 单向数据绑定

我学习Vue的过程中，第一个学到的特殊用法就是 **Mustache 语法** ，但这种方法不能用于**HTML attribute**上，这时候就需要引入利用 `v-bind`  实现的 “单向数据绑定”，“单向数据绑定”很常用也很简单，主要用于绑定**HTML attribute**，以及接受参数。

### 绑定属性

```HTML
<a v-bind:href="url">...</a>
```

```HTML
<a v-bind:[attributeName]="url"> ... </a>
```

### 接受参数

```HTML
<!DOCTYPE html>
<html lang="en">
    <head>
    </head>
    <body>
        <div id="app">
            <test-component r-msg="msg"></test-component>
        </div>

        <script>
            Vue.component('testComponent', {
                props: ['rMsg'],
                template: `
                    <p>
                        {{rMsg}}
                    </p>
                `,
            })

            var app = new Vue({
                el: '#app',
                data: {
                    msg: 'hello',
                },
            })
        </script>
    </body>
</html>

```

父组件 `div` 中的数据通过子组件 `test-component` 中 `prop` 选项中设置的参数 `rMsg` 传入子组件中。

## 双向数据绑定

### `v-model`

双向数据绑定借助 `v-model` 指令实现，其实其本身并不是框架的功能，而是一种语法糖，依附于表单元素，在其上使用时可以实现数据的双向传递。

```HTML
<input v-model="something">
```
与

```HTML
<input v-bind:value="something"  v-on:input="something = $event.target.value">
```

等价。本质上就是忽略所有表单元素的 `value`、`checked`、`selected` attribute 的初始值，而将Vue实例中的数据作为表单元素的 初始值。

`v-model` 在内部为不同的输入元素使用不同的 `property` 并抛出不同的事件：

- `text` 和 `textarea` 元素使用 `value` property 和 `input` 事件；
- `checkbox` 和 `radio` 使用 `checked` property 和 `change` 事件；
- `select` 字段将 `value` 作为 property 并将 `change` 作为事件。

但有些时候，这些特殊的属性我们将它们用于不同的目的，`v-model`这样不分青红皂白地忽略其初始值容易与我们的想法相悖。

这时候就需要引入**自定义组件的 v-model**这一概念了，通过此我们可以自定义`v-model` 使用的 property 和 事件。


```HTML
<!DOCTYPE html>
<html lang="en">
    <head>
    </head>

    <body>
        <div id="app">
            <Child v-model="lovingVue"></Child>
        </div>
        <script>
            // 子组件
            Vue.component('Child', {
                model: {
                    prop: 'num',
                    event: 'change-num',
                },
                props: {
                    num: Number,
                },
                updated: function () {
                    console.log('子组件')
                    console.log(this.num)
                },
                template: `<input type='number' :value='num' @change='$emit("change-num",parseInt($event.target.value))'>`,
            })
            //父组件
            var app = new Vue({
                el: '#app',
                data: {
                    lovingVue: 0,
                },
                updated: function () {
                    console.log('父组件')
                    console.log(this.lovingVue)
                },
            })
        </script>
    </body>
</html>
```

这段代码中子组件 `Child` 通过 `model` 选项设置了 property 和 事件,为什么选项名字叫做 `model` 呢？就是因为这个选项是用来设置 `v-model`的。

设置之后这个组件就成为了一个自定义组件，在这个组件上使用 `v-model` 语法糖时， property 和 事件就都变成了我们设置的，而不是默认的。

事件触发的顺序为：

![image.png](https://b3logfile.com/file/2020/11/image-b85e8ccd.png)

但是在我控制台打印出来的确实下图：

![image.png](https://b3logfile.com/file/2020/11/image-afbdfd4d.png)

---

是我们构想的顺序不对吗？这个问题困扰了两天时间，最终在BB-fat的帮助下找到了可以接受的解释：

`update`函数是在**DOM**元素重新渲染时被调用。我们的`input`元素被包裹在子组件中，子组件又被包裹在父组件中，当我们在`input`中输入时，首先会使子组件重新渲染，子组件渲染完成后然后才使父组件重新渲染。因此被`update` 中先打印出来的是“子组件”。


我们可以用下面的代码验证这个事实：

```HTML
<!DOCTYPE html>
<html lang="en">
    <head>
        <meta charset="UTF-8" />
        <meta name="viewport" content="width=device-width, initial-scale=1.0" />
        <title>Hello</title>
        <script type="text/javascript " src="../assets/js/vue.js"></script>
    </head>

    <body>
        <div id="app">
            <button @click="myClick">click</button>

            <app-son :nums="num"> </app-son>
            <p>{{num}}</p>
        </div>

        <script>
            Vue.component('app-son-son', {
                template: `
                <p>{{numss}}</p>
                `,

                props: ['numss'],
                updated: function () {
                    console.log('子子组件')
                },
            })

            Vue.component('app-son', {
                props: ['nums'],
                template: `
                <div>
                    <p>{{nums}}</p>
                    <app-son-son :numss="nums"></app-son-son>
                </div>
                `,
                updated: function () {
                    console.log('子组件')
                },
            })

            var app = new Vue({
                el: '#app',
                data: {
                    num: 0,
                },
                updated: function () {
                    console.log('父组件')
                },
                methods: {
                    myClick: function () {
                        this.num = this.num + 1
                    },
                },
            })
        </script>
    </body>
</html>

```


执行结果如下，验证了我们的猜想：

![image.png](https://b3logfile.com/file/2020/11/image-37f7b1f6.png)


---

### `.sync`


能达到“双向数据绑定”功能的还有 `.sync` 修饰符,并且咋一看下会觉得他和 `v-model` 没什么区别,事实也确实如此，他们的差别很小，不懂为什么要这么设计两种区别不大的功能。


#### `v-model`
```HTML
<comp v-model="foo"></comp>
               ⇕
<comp :value="foo" @input="v => foo = v"></comp>

this.$emit('input', new_value)
```

#### `v-bind.sync`
```HTML
<comp :bar.sync="foo"></comp>
               ⇕
<comp :bar="foo" @update:bar="v => foo = v"></comp>

this.$emit('update:bar', new_value)
```


我浏览了一些国内外有关这个问题的帖子：

- [What is the difference between v-model and .sync when used on a custom component ?](https://stackoverflow.com/questions/57947757/what-is-the-difference-between-v-model-and-sync-when-used-on-a-custom-component)

- [Vue.js v-model vs. v-bind.sync](https://gist.github.com/AndreKR/80953c53bdd1b3a8dfe0f6f6f29a6020)

- [什么时候用组件的.sync修饰符，什么时候用自定义组件的v-model，两者有什么区别？](https://segmentfault.com/q/1010000014636044)



将大家的意见总结如下：

- 同：都是一种可以实现双向数据绑定的语法糖。
- 异：
1. `v-mdoel` 一般在表单元素上使用，有默认绑定的`property`和 事件；而`.sync`绑定的对象是任意的。
2. 触发 `v-mdoel` 的是组件状态的改变，例如在 `input` 元素上， `change` 事件发生后子组件发出信号；而`.sync` 触发的原因的是变量值的改变。
3. 最主要的区别在于：`v-mdoel` 比较单一，只能出现一次，因此只能绑定一个变量；而`.sync`可以绑定多个变量，如：
    ```HTML
    <comp :value="username" :age="userAge" @update:name="val => userName = val" @update:age="val => userAge = val" />
    ```

至于什么时候采用`v-mdoel`什么时候采用`.sync`？下面的话给出了建议：

> Which one to use? It is a standard pattern to use property value as the key value carrier for a component. In this situation if you have value property and want to enable 2-way binding for it then use v-model. In all other cases use .sync

也就是除了“组件只有`value`这一‘标准’值”的情况，其余情况都应该采用`.sync`。














