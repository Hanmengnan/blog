---
title:  Vue组件写法汇总
tags:  [前端,Vue]
categories: [笔记]
date: 2020/11/14 20:46:25
---
刚开始接触前端，第一个接触的框架是**Vue**，众所周知“数据驱动”和“组件化”是vue.js两个最重要的特点。组件化是为了方便代码复用，提高开发效率。常见的vue组件写法有四种，各有特色，适用于不同的场景。

入门时已经接触到了“组件”的概念，但是当时看到的组件写法，和第一份项目代码中的写法完全不同，所以当时很困惑，所以收集了一下Vue中“组件”的不同写法在此。


### `script`标签引入

#### 全局组件
```Vue
<body>
    <div id="app">
        <componentName></componentName>
    </div>
    <script>
        Vue.component('componentName', {
            data() {
                return {
                    message: "hello",
                }
            },
            
            method: {

            },
            // 组件中的方法

            //......
            // 组件其他的属性和方法 

            template: "<div></div>",
            // 组件的html结构, 

        })

        new Vue({
            el: '#app'
        })
    </script>
</body>
```

- 这样注册的组件，可以被`script`标签中的其他Vue实例使用。
- 但缺点是因为是全局组件，所以使用的时候要注意名字唯一。

#### 局部组件

```Vue  
<body>
    <div id="app">
        <componentName></componentName>
    </div>
    <script>
        var myComponent = {
            data() {
                return {
                    message: "hello",
                }
            },
            
            method: {

            },
            // 组件中的方法

            //......
            // 组件其他的属性和方法 

            template: "<div></div>",
            // 组件的html结构, 

        }

        new Vue({
            el: '#app',
            components: {
                myComponent,
            }
        })
    </script>
</body>
```
- 这样的组件，只有在Vue实例中注册过才能使用。


### `template`标签引入

#### 全局组件
```Vue
<body>
    <template id="my-component">
        hello
    </template>


    <div id="app">
        <componentName></componentName>
    </div>
    <script>
        Vue.component({
            data() {
                return {
                    message: "hello",
                }
            },

            method: {

            },
            // 组件中的方法

            //......
            // 组件其他的属性和方法 

            template: "#myComponent",
            // 组件的html结构, 

        })

        new Vue({
            el: '#app',
        })
    </script>
</body>
```

#### 局部组件
```Vue
<body>
    <template id="my-component">
        hello
    </template>


    <div id="app">
        <componentName></componentName>
    </div>
    <script>
        var myComponent = {
            data() {
                return {
                    message: "hello",
                }
            },

            method: {

            },
            // 组件中的方法

            //......
            // 组件其他的属性和方法 

            template: "#myComponent",
            // 组件的html结构, 

        }

        new Vue({
            el: '#app',
            components: {
                myComponent,
            }
        })
    </script>
</body>
```

### 单文件组件

```Vue
<template id="component-test">
    <div>
        <h1>test</h1>
        <xxxx></xxxx>
    </div>
</template>

<script>
import xxxx from '../xxxx.min.js'
// .....
// 组件逻辑

export default {
    name: 'myComponent',
    components: {
        xxxx,
    },
    data() {
        return {
            messgae: 'hello',
        }
    },
    methods: {},
    computed: {},

    // .....
    // 组件属性和方法
}
</script>

<style></style>

```
- 采用该种写法时，常常在项目目录下新建一个`component`目录，目录下放置“.vue”结尾的文件，文件名即组件名，一个文件对应一个组件。

- 文件按照“template”；“script”；“style”标签的顺序组织。

- 这样组件与组件之间互不影响，复用性高，其html、css、js均可复用。组件的结构、逻辑清晰。因此这个方式是代码量较大的项目中常用的组件写法。


