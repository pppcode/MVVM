# MVVM 框架

## 项目介绍

原生 JS 实现一个 MVVM 框架（Vue 的核心）

效果：

- 页面内容就是 data 中的数据
- 修改内容时，会有提示（事件的绑定，点击的时候调用'sayHi' , 也就是 methods 中的方法，再去调用 data 中的数据）
- 双向绑定：修改内容时，页面显示的内容也跟着发生变化

## 具体实现

**流程**

1. 数据的劫持：使用 Object.defineProperty 实现数据的拦截
2. 发布订阅模式：拦截数据后，在合适的时间去订阅数据的变动，在数据变动时再去触发修改

### Object.defineProperty 的用法

Object.defineProperty() 方法会直接在一个对象上定义一个新属性，或者修改一个对象的现有属性。这个方法执行完成后会返回修改后的这个对象。

**语法**

`Object.defineProperty(obj, prop, descriptor)`

- obj：要处理的目标对象
- prop：要定义或修改的属性名
- descriptor：将被定义或修改的属性描述符

**使用方法**

```
var obj = {}
obj.name = 'zhangsan'
obj['age'] = 3
//对对象里面的属性和值操作的方法（一种新的）
Object.defineProperty(obj, 'intro', {
  value: 'hello world'
})
console.log(obj) //{name: "zhangsan", age: 3, intro: "hello world"}
```

以上三种方法都可以用来定义/修改一个属性，Object.defineProperty 的方法貌似有些小题大做，没关系，且往下看他更富在的用法

```
var obj = {}
Object.defineProperty(obj, 'intro', {
  configurable: false,
  value: 'hello world'
})
obj.intro = 'zhangsan'
console.log(obj.intro) //'hello world'
delete obj.intro //false,删除失败
console.log(obj.intro) //'hello world'

Object.defineProperty(obj, 'intro', {
  configurable: true,
  value: 'hello world'
})
delete obj.intro //true,删除成功
console.log(obj.intro) //undefined
```

通过上面的例子可以看出，属性描述对象中的 configurable 的值设为 false 后（没设置的话，默认为 false），以后就不能修改属性，也无法删除该属性

```
var obj = {name: 'zhangsan'}
Object.defineProperty(obj, 'age', {
  value: 3,
  enumerable: false
})
for(var key in obj) {
  console.log(key) //只输出 'name',不输出 'age'
}
```

设置 enumerable 属性为 false 后，遍历对象的时候回忽略通过 defineProperty 设置的属性（如果未设置，默认就是 false 不可遍历）,设为 true 后，对象中的属性全部可遍历。

```
var obj = {name: 'zhangsan'}
Object.defineProperty(obj, 'age', {
  value: 3,
  writable: false
})
obj.age = 4
console.log(obj.age) //3
```

设置 writable 为 false 时，修改对象的 defineProperty 中设置的属性值无效。

value 和 writable 叫数据描述符，具有以下可选键值：

- value: 该属性对应的值，可以是任何有效的 JavaScript 值（数值，对象，函数等），默认为 undefined.
- writable: 当且仅当该属性的 writable 为 true 时，该属性才能被赋值运算符改变，默认为 false.

configurable: true 和 writable: true 的区别是，前者是设置属性能删除， 后者是设置属性能修改。

```
var obj = {}
var age
Object.defineProperty(obj, 'age', {
  get: function() {
    console.log('get age ...')
    return age
  },
  set: function(val) {
    console.log('set age ...')
    age = val
  }
})
obj.age = 200 //'set age ...' age = val = 200,此处的 age 是值，等于 get 中的 age
obj.age //'get age ...' 200 
```

get 和 set 叫存取描述符，有以下可键值：

- get: 一个给属性提供 getter 的方法，如果没有 getter 则为 undefined。改方法返回值被用作属性值，默认为 undefined。

```
//没有 getter 
var obj = {}
Object.defineProperty(obj, 'age', {})
obj.age //undefined

//getter 的返回值被用做属性值，默认为 undefined
var obj = {}
var age 
Object.defineProperty(obj, 'age', {
  get: function() {
    console.log('get age ...')
    return age
  }
})
obj.age //undefined 先从里面找，没有的话，再去全局找到 age = undefined
```
- set: 一个给属性提供 setter 的方法，如果没有 setter 则为 undefined。该方法将接受唯一参数，并将该参数的新值分配给该属性，默认为 undefined。

数据描述符和存取描述符是不能同时存在的

```
var obj = {}
var age 
Object.defineProperty(obj, 'age', {
  value: 100,
  get: function() {
    console.log('get age ...')
    return age 
  },
  set: function(val) {
    console.log('set age ...')
    age = val
  }
})
```

上输代码会报错，value/writable 不能和 get/set 同时存在,因为这些都是对数据的描述,所以这种写法是不行的。

**通过 get 和 set 可以做数据的劫持了**

- 当用户通过一个对象调用它的属性得到它的值得时候，中间就会被拦截了一道，调用 get ，
- 当用户去设置这个值得时候，中间也会变被拦截，调用 set。

### 数据的劫持

现在我们利用 Object.defineProperty 这个方法动态监听数据,data 里的数据被监听了，当用户操作数据时，observe() 会去做对应的事情，这就实现了数据的拦截

```
//定义数据
var data = {
  name: 'zhangsan',
  friends: [1, 2, 3]
}

//定义监听函数
function observe(data) {
  if(!data || typeof data !== 'object') return 
  for(var key in data) {
    //遍历时，得到每个属性的值，延伸：1.这里是 let 而不是 var ，为什么
    let val = data[key]
    Object.defineProperty(data, key, {
      enumerable: true, //可遍历
      configurable: true, //可删除
      get: function() {
        console.log(`get ${val}`)
        return val //延伸：2.val 能否替换成 data[key]
      },
      //调用set 时， 把值作为参数传递进来，把新值 赋值给 旧值
      set: function(newVal) {
        console.log(`changes happen: ${val} => ${newVal}`)
        val = newVal
      }
    })
    //如果值是对象，再次执行 obsrve ，递归操作，递归到了一定的层级，传递的值不是对象，就会停掉
    if(typeof val === 'object') {
      observe(val) 
    }
  }
}

//监听数据
observe(data)

//操作数据
console.log(data.name) //'get zhangsan' zhangsan
data.name = 'lisi' //'changes happen: zhangsan => lisi' lisi
data.friends[0] //'get 1' 1
data.friends[0] = 4 //'changes happen: 1 => 4' 4
```

上面的 observe 函数实现了一个数据监听，当监听某个对象后，我们可以在用户读取或者设置属性值的时候做个拦截，做我们想做的事，**所做的事情都在 get 函数里，做不了什么复杂的事情**

延伸

1. 为什么用 let 而不是 var ？

let 有块级作用域 {},每次遍历，都能存贮当前的状态，每次的 val 是不一样的，当用户调用 get 时，获取的是当前作用域下的 val，如果用 var 的话，遍历完成后，val 是最后一次的值，用户再次获取 val 的值是都是获取的最后一次的值,`data.name //get 3 `,不符合需求。

若用 var 的话，则需要这样写, 利用 forEach 提供的回调函数

```
function observe(data) {
  if (!data || typeof data !== 'object') return
  var keys = Object.keys(data)
  keys.forEach(key => {
    //每个 val 是不一样的， 因为他处于这个回调函数里面
    var val = data[key]
    Object.defineProperty(data, key, {
      enumerable: true,
      configurable: true,
      get: function () {
        console.log(`get ${val}`)
        return val
      },
      set: function (newVal) {
        console.log(`changes happen: ${val} => ${newVal}`)
        val = newVal
      }
    })
    if (typeof val === 'object') {
      observe(val)
    }
  })
}
```

或者用闭包

```
function observe(data) {
  if (!data || typeof data !== 'object') return
  for (let key in data) {
    //闭包，用立即执行函数存贮每次遍历的 key
    (function (key) {
      var val = data[key]
      Object.defineProperty(data, key, {
        enumerable: true,
        configurable: true,
        get: function () {
          console.log(`get ${val}`)
          return val
        },
        set: function (newVal) {
          console.log(`changes happen: ${val} => ${newVal}`)
          val = newVal
        }
      })
      if (typeof val === 'object') {
        observe(val)
      }
    })(key)
  }
}
```
操作数据`data.name //'get zhangsan' zhangsan`，均符合需求。

2. get 中的 val 能否替换成 data[key]？

不可以，会陷入死循环，例如当用户调用 data[name] 执行 get()， 执行的过程中又需要 data[name],当需要得到 data[name] 的值时，又需要调用 get(),调用get() 时，又需要...

### 观察者模式

一个典型的观察者模式应用场景是用户订阅某个主题

1. 多个用户（观察者，Observer）都可以订阅某个主题（Subject）
2. 当主题内容更新时订阅该主题的用户都能收到通知

Subject 是构造函数，new Subject() 创建一个主体对象，该对象内部维护订阅当前主题的观察者数组。主题对象上有一些方法，如添加观察者 (addObserver),删除观察者（removeObserver）,通知观察者更新（notify）, 当 notify 时实际上调用全部观察者 observer 自身的 update 方法

Observer 是构造函数， new Observer() 创建一个观察者对象，该对象有一个 update 方法。

```
function Subject() {
  this.observers= []
}
Subject.prototype.addObserver = function(observer) {
  this.observers.push(observer)
}
Subject.prototype.removeObserver = function(observer) {
  var index = this.observers.indexOf(observer)
  if(index > -1) {
    this.observers.splice(index, 1)
  }
}
Subject.prototype.notify = function() {
  this.observers.forEach(function(observer){
    observer.update()
  })
}

function Observer(name) {
  this.name = name
  this.update = function() {
    console.log(name + ' update...')
  }
}

//创建主题
var subject = new Subject()
//创建观察者1
var observer1 = new Observer('zhangsan')
//主题添加观察者1
subject.addObserver(observer1)
//创建观察者2
var observer2 = new Observer('lisi')
//主题添加观察者2
subject.addObserver(observer2)
//主题通知所有观察者更新
subject.notify()
```

**ES6 写法**

```
class Subject {
  constructor() {
    this.observers = []
  }
  addObserver(observer) {
    this.observers.push(observer)
  }
  removeObserver(observer) {
    var index = this.observers.indexOf(observer)
    if(index > -1) {
      this.observers.splice(index, 1)
    }
  }
  notify() {
    this.observers.forEach(observer => {
      observer.update()
    })
  }
}

class Observer {
  constructor(name) {
    this.name = name
  }
  update() {
    console.log(this.name + ' update...')
  }
}

let subject = new Subject()
let observer1 = new Observer('zhangsan')
subject.addObserver(observer1)
let observer2 = new Observer('lisi')
subject.addObserver(observer2)

subject.notify()
```
**有个小问题**

用户订阅博客，是用户来操作，用户发起的，而不是主题去发起的，做一些修改

```
class Observer {
  constructor(name) {
    this.name = name
  }
  update() {
    console.log(this.name + ' update...')
  }
  subscribeTo(subject) {
    subject.addObserver(this)
    console.log('订阅成功！')
  }
}

let subject = new Subject()
let observer = new Observer('zhangsan')
observer.subscribeTo(subject)

subject.notify()
```

增加 subscribeTO 方法，我是观察者，我要订阅你，当订阅 subject 时，调用 subject.addObserver()，把我加进去，其实就是加了一层壳
从外部调用来看，更符合逻辑，更直观了。

**总结**

我去订阅某个主题，可以订阅不同的主题，当主题发出通知后，我就能收到，执行我的方法去更新。

### MVVM 单项绑定的实现

1. 解析并编译模板，字符串替换成数据中对应的值。

    首先进行初始化，数据暂存起来，执行 compile() 编译模板，遍历节点，如果有子元素的话，再去遍历子元素，直到里面的内容是文本，开始处理文本，把文本做个替换: 正则匹配模板中的{{name}},{{age}},一个个匹配，匹配完后，把里面的 name 换成 以 name 作为属性的值，整体会被换掉，此时用户看到的就是真实的数据了。

```
<div id="app">
  <h1>{{name}} 's is {{age}}</h1>
</div>

<script>
  class mvvm {
    constructor(opts) {
      this.init(opts)
      this.compile()
    }
    //初始化，数据暂存起来
    init(opts) {
      this.$el = document.querySelector(opts.el)
      this.$data = opts.data
    }
    //对模板进行解析
    compile() {
      this.traverse(this.$el)
    }
    //遍历模板，可能有多层，需要用到递归
    traverse(node) {
      if(node.nodeType === 1) { //div
        node.childNodes.forEach(childNode => {
          this.traverse(childNode) //递归
        })
      }else if(node.nodeType === 3) { //文本
        this.renderText(node)
      }
    }
    //渲染模板，替换成真实的数据
    renderText(node) {
      let reg = /{{(.+?)}}/g
      let match
      while(match = reg.exec(node.nodeValue)) { //循环两次，拿到 [{{name}}, name], [{{age}}, age]
        let raw = match[0] //{{name}} {{age}}
        let key = match[1].trim() //name age,删除字符串两端的空白字符
        node.nodeValue = node.nodeValue.replace(raw, this.$data[key]) //字符串替换成对应的数据
      }
    }
  }

  let vm = new mvvm({
    el: '#app',
    data: {
      name: 'zhangsan',
      age: 3
    }
  })
</script>
```

页面显示：由`{{name}} 's is {{age}}`变为`zhangsan 's is 3`。

问题：用户修改数据后，模板中的 {{name}},{{age}} 怎么进行实时的变动呢？










