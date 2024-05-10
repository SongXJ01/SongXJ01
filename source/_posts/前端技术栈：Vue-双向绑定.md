---
title: 前端技术栈：Vue 双向绑定
date: 2021-08-17 13:59:02
tags: [Vue, 前端]
categories:
- 技术笔记
- 前端技术栈
---



## MVVM模式

&emsp;&emsp;说到 Vue 的双向绑定首先联系到的就是 MVVM（Model-View-ViewModel）模式了，如下图所示，当视图发生改变的时候传递给 VM，再让数据得到更新，当数据发生改变的时候传给 VM，使得视图发生改变。

<!--more-->

MVVM 模式是通过以下三个核心组件组成：
 - **M：** Model - 包含了业务和验证逻辑的数据模型；
 - **V：** View - 定义屏幕中 View 的结构，布局和外观；
 - **VM：** ViewModel - 扮演“View”和“Model”之间的使者，帮忙处理 View 的全部业务逻辑。

![vue数据双向绑定原理](/SongXJ01/images/前端技术栈_Vue双向绑定/vue数据双向绑定原理.png)



-----



## Vue 数据双向绑定原理
&emsp;&emsp;Vue 数据双向绑定是通过**数据劫持结合发布者-订阅者**模式的方式来实现的，那么 Vue 是如果进行数据劫持的。我们可以先来看一下通过控制台输出一个定义在 Vue 初始化数据上的对象是个什么东西。

```javascript
var vm = new Vue({
    data: {
        obj: {
            a: 1
        }
    },
    created: function () {
        console.log(this.obj);
    }
});
```
输出：
![输出](/SongXJ01/images/前端技术栈_Vue双向绑定/输出.png)


可以看到属性`a`有两个相对应的`get`和`set`方法，为什么会多出这两个方法呢？因为 Vue 是通过 `Object.defineProperty()` 来实现数据劫持的。


### 通过一个“加《XXX》”的例子来理解
&emsp;&emsp;在平常，很容易就可以打印出一个对象的属性数据：

```javascript
var Book = {
  name: 'vue权威指南'
};
console.log(Book.name);  // vue权威指南
```
&emsp;&emsp;如果想要在执行`console.log(book.name)`的同时，直接给书名加个书名号，那要怎么处理呢？或者说要通过什么监听对象 `Book` 的属性值。这时候`Object.defineProperty( )`就派上用场了，代码如下：

```javascript
var Book = {}
var name = '';
Object.defineProperty(Book, 'name', {
  set: function (value) {
    name = value;
    console.log('书名叫做' + value);
  },
  get: function () {
    return '《' + name + '》'
  }
})
 
Book.name = 'vue权威指南';  // 书名叫做vue权威指南
console.log(Book.name);  // 《vue权威指南》
```

&emsp;&emsp;通过`Object.defineProperty( )`设置了对象 `Book` 的 `name` 属性，对其 `get` 和 `set` 进行重写操作，顾名思义，`get` 就是在读取 `name` 属性这个值触发的函数，`set` 就是在设置 `name` 属性这个值触发的函数，所以当执行 `Book.name = 'vue权威指南'` 这个语句时，控制台会打印出 `"书名叫做vue权威指南"`，紧接着，当读取这个属性时，就会输出 `"《vue权威指南》"`，因为我们在 `get` 函数里面对该值做了加工。


### 思路分析
&emsp;&emsp;实现 **MVVM**主要包含两个方面，数据变化更新视图，视图变化更新数据：
![MVVM](/SongXJ01/images/前端技术栈_Vue双向绑定/MVVM.png)

&emsp;&emsp;关键点在于 data 如何更新 view，因为 view 更新 data 其实可以通过事件监听即可，比如 input 标签监听 input 事件就可以实现了。

&emsp;&emsp;数据更新视图的重点是如何知道数据变了，只要知道数据变了，那么接下去的事都好处理。如何知道数据变了，其实上文我们已经给出答案了，就是通过`Object.defineProperty( )`对属性设置一个 set 函数，当数据改变了就会来触发这个函数，所以我们只要将一些需要更新的方法放在这里面就可以实现 data 更新 view 了。
![defineProperty](/SongXJ01/images/前端技术栈_Vue双向绑定/defineProperty.png)


----

## 实现双向绑定
&emsp;&emsp;首先要对数据进行劫持监听，所以我们需要设置一个**监听器 Observer**，用来监听所有属性。如果属性发生变化了，就需要告诉**订阅者 Watcher** 看是否需要更新。因为订阅者是有很多个，所以我们需要有一个**消息订阅器 Dep** 来专门收集这些订阅者，然后在监听器Observer 和 订阅者 Watcher 之间进行统一管理的。接着，还需要有一个**指令解析器 Compile**，对每个节点元素进行扫描和解析，将相关指令对应初始化成一个订阅者 Watcher，并替换模板数据或者绑定相应的函数，此时当订阅者 Watcher 接收到相应属性的变化，就会执行对应的更新函数，从而更新视图。

**双向绑定步骤：**

1. 实现一个**监听器 Observer**，用来劫持并监听所有属性，如果有变动的，就通知订阅者。

2. 实现一个**订阅者 Watcher**，可以收到属性的变化通知并执行相应的函数，从而更新视图。

3. 实现一个**解析器 Compile**，可以扫描和解析每个节点的相关指令，并根据初始化模板数据以及初始化相应的订阅器。

**流程图如下：**
![双向绑定流程图](/SongXJ01/images/前端技术栈_Vue双向绑定/双向绑定流程图.png)


-----

## 实现最简单的双向绑定

```html
<!doctype html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Document</title>
</head>
<body>
    <div id="demo"></div>
    <input type="text" id="inp">
    <script>
        var obj  = {};
        var demo = document.querySelector('#demo')
        var inp = document.querySelector('#inp')
        Object.defineProperty(obj, 'name', {
            get: function() {
                return val;
            },
            set: function (newVal) {// 当该属性被赋值的时候触发
                inp.value = newVal;
                demo.innerHTML = newVal;
            }
        })
        inp.addEventListener('input', function(e) {
            // 给obj的name属性赋值，进而触发该属性的set方法
            obj.name = e.target.value;
        });
        obj.name = 'fei';// 在给obj设置name属性的时候，触发了set这个方法
    </script>
</body>
</html>
```


----

## Vue 代码实现
### 1. 实现 observer
&emsp;&emsp;主要是给每个vue的属性用Object.defineProperty()，代码如下：


```javascript
function defineReactive (obj, key, val) {
    var dep = new Dep();
        Object.defineProperty(obj, key, {
             get: function() {
                    //添加订阅者watcher到主题对象Dep
                    if(Dep.target) {
                        // JS的浏览器单线程特性，保证这个全局变量在同一时间内，只会有同一个监听器使用
                        dep.addSub(Dep.target);
                    }
                    return val;
             },
             set: function (newVal) {
                    if(newVal === val) return;
                    val = newVal;
                    console.log(val);
                    // 作为发布者发出通知
                    dep.notify();//通知后dep会循环调用各自的update方法更新视图
             }
       })
}
        function observe(obj, vm) {
            Object.keys(obj).forEach(function(key) {
                defineReactive(vm, key, obj[key]);
            })
        }
```


### 2. 实现 compile

&emsp;&emsp;compile 的目的就是解析各种指令称真正的 html。


```javascript
function Compile(node, vm) {
    if(node) {
        this.$frag = this.nodeToFragment(node, vm);
        return this.$frag;
    }
}
Compile.prototype = {
    nodeToFragment: function(node, vm) {
        var self = this;
        var frag = document.createDocumentFragment();
        var child;
        while(child = node.firstChild) {
            console.log([child])
            self.compileElement(child, vm);
            frag.append(child); // 将所有子节点添加到fragment中
        }
        return frag;
    },
    compileElement: function(node, vm) {
        var reg = /\{\{(.*)\}\}/;
        //节点类型为元素(input元素这里)
        if(node.nodeType === 1) {
            var attr = node.attributes;
            // 解析属性
            for(var i = 0; i < attr.length; i++ ) {
                if(attr[i].nodeName == 'v-model') {//遍历属性节点找到v-model的属性
                    var name = attr[i].nodeValue; // 获取v-model绑定的属性名
                    node.addEventListener('input', function(e) {
                        // 给相应的data属性赋值，进而触发该属性的set方法
                        vm[name]= e.target.value;
                    });
                    new Watcher(vm, node, name, 'value');//创建新的watcher，会触发函数向对应属性的dep数组中添加订阅者，
                }
            };
        }
        //节点类型为text
        if(node.nodeType === 3) {
            if(reg.test(node.nodeValue)) {
                var name = RegExp.$1; // 获取匹配到的字符串
                name = name.trim();
                new Watcher(vm, node, name, 'nodeValue');
            }
        }
    }
}
```


### 3. 实现 watcher


```javascript
function Watcher(vm, node, name, type) {
    Dep.target = this;
    this.name = name;
    this.node = node;
    this.vm = vm;
    this.type = type;
    this.update();
    Dep.target = null;
}

Watcher.prototype = {
    update: function() {
        this.get();
        this.node[this.type] = this.value; // 订阅者执行相应操作
    },
    // 获取data的属性值
    get: function() {
        console.log(1)
        this.value = this.vm[this.name]; //触发相应属性的get
    }
}
```


### 4. 实现Dep来为每个属性添加订阅者



```javascript
function Dep() {
    this.subs = [];
}
Dep.prototype = {
    addSub: function(sub) {
        this.subs.push(sub);
    },
    notify: function() {
        this.subs.forEach(function(sub) {
        sub.update();
        })
    }
}
```




----

## 总结
&emsp;&emsp;首先，为每个 Vue 属性用 `Object.defineProperty()` 实现数据劫持，为每个属性分配一个订阅者集合的管理数组 dep；然后在编译的时候在该属性的数组 dep 中添加订阅者，`v-model` 会添加一个订阅者，`{{}}` 也会，`v-bind` 也会，只要用到该属性的指令理论上都会，接着为 input 会添加监听事件，修改值就会为该属性赋值，触发该属性的 set 方法，在 set 方法内通知订阅者数组 dep，订阅者数组循环调用各订阅者的 `update` 方法更新视图。

### v-model
&emsp;&emsp;v-model 虽然很像使用了双向数据绑定的 Angular 的 ng-model，但是 Vue 是单项数据流，v-model 只是语法糖而已。

&emsp;&emsp;第一行的代码其实只是第二行的语法糖。
```html
<input v-model="sth" />
<input v-bind:value="sth" v-on:input="sth = $event.target.value" />
```

<br/><br/>


----
参考来源：
1. https://www.cnblogs.com/chenhuichao/p/10818396.html
2. https://www.jianshu.com/p/5fe2664ff5f7


<br/><br/><br/><br/>