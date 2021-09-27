# 感受Proxy的魅力-深拷贝(小白看得懂系列)

## 背景
最近找下深拷贝的资料，阅读了一些资料，看得不太清楚，大部分人云亦云，学习最主要抱着学精的态度，不怕慢 最主要能学进大脑
故整理了一些心得经验，就有此文，选择了个人认为最好的一种深拷贝方式`proxy`（如果你有更好的办法 欢迎不吝指教）

本文描述以最基础开始，尽量直白简单化，带领你走入`Proxy`的第一步，从而也能够认识`Proxy`的魅力

## 开始

什么时候用到深拷贝？你也许会遇到过这种 后台返回了一堆`Object`,你需要按照设定好的算法 去返回某部分的对象,但是又不想破坏掉原对象
所以你可能会通过
* `JSON.stringify(Object)`把原对象转成字符串,但是会有很多局限性
* 另外一个就比较复杂, 可以查看[Lodash](https://www.lodashjs.com/docs/lodash.cloneDeep)的实现方式

先来看看 我们需要浅拷贝怎么样做? 看一段例子
对象是引用类型,顾名思义,也就是互相引用,所以我在赋值进行修改,会影响原来的对象
```JavaScript
//对象 
let o = {a:1}
let _copyO = o
_copyO.a = 2
o.a // 2

//数组
let arr = ['a'] 
let _copyArr = a 
_copyArr = ['b']
arr // ['b']
```

所以这里可以用浅拷贝的方式，能很简单的达到我们的目的

```JavaScript
//对象 
let o = {a:1}
let _copyO = {...o}
_copyO.a = 2
o.a // 1

//数组
let arr = ['a'] 
let _copyArr = arr.slice()
_copyArr = ['b']

arr  // ['a']

```
这里核心的拷贝原理还是基于最简单粗暴的：

* 对象通过[扩张运算符](https://es6.ruanyifeng.com/#docs/object#%E5%AF%B9%E8%B1%A1%E7%9A%84%E6%89%A9%E5%B1%95%E8%BF%90%E7%AE%97%E7%AC%A6)的方式
* 数组通过`slice()`的方法
> slice() 方法可从已有的数组中返回选定的元素。**不会改变原始数组**

## 浅拷贝

我们知道了原理之后, 就可以通过`Proxy`来实现浅拷贝
> 如果你不太了解`Proxy`的用法，建议先去[MDN刷一遍](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Proxy)

思考: 为什么要用`Proxy`去实现拷贝?
我明明只修改了这个属性，为什么不能够只对这部分的属性做拷贝呢？ 这里就是`Proxy`代理的魅力了。

> 当然你也可以用`Object.defineProperty`,但是这只能对某个属性改变了而作出响应，而`Proxy`是代理了当前的对象

先走一个浅拷贝
```javaScript
const people = {
  man: {
    name: 'lee',
    body: {
      foot: {
        state: 'good'
      },
      hand: {
        state: 'good'
      }
    }
  },
  friend: ['jw', 'gg', 'gw']
}
  
const copyMap = new Map() 
const _handle = {
  get(target, prop) {},
  set(target, prop, value) {
    const _cope = Array.isArray(target) ? target.slice() : {...target} // 复制数据
    _cope[prop] = value
      // 这里很重要 保存 复制后的数据 方便调取
    copyMap.set(target, _cope)
    
    return true
  }
}
let _py = new Proxy(people, _handle)
_py.friend = ['jw']

let p = copyMap.get(people)
console.log(p)
console.log(people)
```
运行上面代码, 可以看到已经被拷贝,不会影响了原对象
运行结果
```
p => {man: {…}, friend: ["jw"]}
people => {man: {…}, friend: ["jw", "gg", "gw"]}
```


在这里我想修改`man`这个属性
```javascript
_py.friend = ['jw']
_py.man = {}

let p = copyMap.get(people)
console.log(p)
console.log(people)
```

然后在看看输出
```
{
friend:["jw", "gg", "gw"]
man: {}
}
```
`man`这个属性被修改了，而`friend`这个属性没有变化 
原因是因为 `copyMap.set(target, _cope)` 永远就更新了最后一个, 所以通过`copyMap`根据原对象取值的时候, 就是最后一个的变化
只需要进行一些改动
```javaScript
const copyMap = new Map()
const _handle = {
  get(target, prop) {},
  set(target, prop, value) {
    let _copy
    // 这里如果被保存过在copyMap里面 就不要在另外复制一个 而是用当前这个进行再次修改
    if (copyMap.has(target)) {
      _copy = copyMap.get(target)
    } else {
      _copy = Array.isArray(target) ? target.slice() : { ...target } // 复制数据
    }
    _copy[prop] = value
    // 保存每一段修改后的数据
    copyMap.set(target, _copy)
    return true
  }
}

let _py = new Proxy(people, _handle)
_py.man = {}
_py.friend = ['jw']
let p = copyMap.get(people)
console.log(p)
console.log(people)
```
原理就是，把拷贝的数据存到`new Map()`,更新的属性反响到拷贝的数据。最后就取拷贝好的对象就可以。
这就是用`proxy`来处理浅拷贝的原理，只对更新的属性这部分做出相应，性能达到最大化

## 深拷贝

了解了浅拷贝之后，就可以看看重点深拷贝拉

还是沿用上面的例子，我们来添加多一级的属性修改`_py.man.name = 'xiaoMing'`
会发现有报错
```
Uncaught TypeError: Cannot set property 'name' of undefined
```

因为根本没有进入到`set()`这个捕捉器，我理解是，这条链子太长了，手够不着，只能获取到最后一个属性更新的value
但是`get()`这个捕捉器刚好相反，你够不着，我都能够摸得到。并能获得每一次属性的值
上个例子
```javaScript
const copyMap = new Map()
const proxyMap = new Map()

const _handle = {
  get(target, prop) {
    console.log('get捕捉器', target, prop)
    const _data = copyMap.get(target) || target
    const _proxy = setProxy(_data[prop])
    return _proxy
  },
  set(target, prop, value) {
    console.log('set捕捉器', target, prop, value)
    let _copy  
    if (copyMap.has(target)) {
      _copy = copyMap.get(target)
    } else {
      _copy = Array.isArray(target) ? target.slice() : { ...target } // 复制数据
    }
    _copy[prop] = value
    copyMap.set(target, _copy)
    return true
  }
}

// 把转换成proxy的复用 写成一个函数去处理
const setProxy = (target) => {
  // 检测是不是普通对象 或者 是数组
  if (Object.prototype.toString.call(target) === '[object Object]' || Array.isArray(target)) {
    if (proxyMap.has(target)) {
      return proxyMap.get(target)
    }
    const _proxy = new Proxy(target, _handle)
    proxyMap.set(target, _proxy)
    return _proxy
  }
  return target
}

let _py = setProxy(people)
_py.man.name = 'xiaoMing'
let p = copyMap.get(people)
console.log(p) // undefined 
```
需要在`get`捕获器去处理，每次进来的属性值 都转成对应的`proxy`对象
这里新增一个`setProxy()`方法,需要复用相同的转换`proxy`的方法
* 定义了一个`proxyMap`的变量,是`Map`类型
* 每次新增都保存到`proxyMap`的栈中
* 如果已在`proxyMap`的栈中匹配到相同的，直接取出来使用

看看上面的结果, 输出的是`undefined`,因为`copyMap`保存的是`man`这个拷贝对象,并是已经做了拷贝赋值的对象`man.name = 'xiaoMing'` 

在回头看看`Proxy`的核心，我只对操作的这部分属性做出相应，所以就只保存这部分做出相应内容

### 来个看形容点情景

你做出了一盘好看的水果盘，里面都是你喜欢吃的水果，其他商家抄袭了你的水果盘

那么你就继续创新去变化更好看的水果盘，可恶的商家也跟着变

有办法了！ 我把需要变化的部分单独保存，最后在拼接成另外一个水果盘 

我只需要关系我需要变动的这部分，也减少了我的工作量
> 大概是这个意思，绞尽脑汁才想出来的


回到例子，所以这里的而`people`对象没有被保存，只保存了需要**变化的部分**

那么就要把`people`这个原始对象丢进去处理，拷贝一份出来
这里写多一个函数处理Map
```javaScript
const _handleMap = (target) => {
  // 只处理普通对象 和 数组  因为是递归循环 最后肯定会获得单个数值的 所以就直接只处理对象或者数组就可以了
  if (Object.prototype.toString.call(target) === '[object Object]' || Array.isArray(target)) {
    let _copy
    const copy = Array.isArray(target) ? target.slice() : { ...target } // 复制数据
    // 这里处理copy的缓存
    if (copyMap.has(target)) {
      _copy = copyMap.get(target)
    } else {
      copyMap.set(target, copy)
    }

    const _co = copyMap.get(target)
    // **这里是重点
    // 递归循环对象的每一个键值 然后在copyMap里取出来 赋值回去
    // 这里就可以做到深拷贝拉
    Object.keys(_co).forEach((item) => {
      _co[item] = _handleMap(_co[item])
    })
    return _co
  }
  // 其他类型就直接返回 比如字符串 函数 数值什么的
  return target
}

let _py = setProxy(people)
_py.man.name = 'xiaoMing'
_handleMap(target)

let p = copyMap.get(people)
```
也许你看到这里有点懵，但是这是整合深拷贝的最后一个核心方法，下面再也没其他操作拉
先来解析看看
* 把`people`丢到函数进行加工处理，得到一个拷贝的对象，并存到`copyMap`里面
* 开始循环`people`对象的key值,然后根据key匹配到对应的**变化的部分**在添加回去
* 最终就构建了一个完整 拷贝对象，根据原对象去获取就可以

整个深拷贝到这里结束

## 优化

你会发现, `copyMap`为什么连`body`下的属性也添加了进去，不是说了 只操作变化的部分吗？
那这没区别啊？还是要循环整个对象

所以这里得加多一行代码 去进行处理
```javaScript
const _handleMap = (target) => {
if (Object.prototype.toString.call(target) === '[object Object]' || Array.isArray(target)) {
  // 没有被map保存过的，也就是肯定没有被操作过的
  if (!(proxyMap.has(target) || copyMap.has(target))) {
    return target
  }
  let _copy
  const copy = Array.isArray(target) ? target.slice() : { ...target } // 
  if (copyMap.has(target)) {
    _copy = copyMap.get(target)
  } else {
    copyMap.set(target, copy)
  }

  const _co = copyMap.get(target)

  Object.keys(_co).forEach((item) => {
    _co[item] = _handleMap(_co[item])
  })
  return _co
}
// 其他类型就直接返回 比如字符串 函数 数值什么的
return target
}
```
这样就可以提前拦截，没有被操作的属性直接忽略，不去执行递归，从而达到了性能最大化

最后还可以优化一下代码，把复用的代码抽出来，比如` const copy = Array.isArray(target) ? target.slice() : { ...target } `这一段的代码逻辑
最终还需要封装成函数对外调用

源码地址:(配合源码食用更合适)
[https://github.com/Power-kxLee/proxyCopyCode](https://github.com/Power-kxLee/proxyCopyCode)

## 总结
* 这是我见过最高性能的深拷贝, 只对变化的这一部分进行操作
* 所以需要知道操作得是那部分，`Prxoy`在这里完全是C位
* 最终整合变化的部分，拷贝一个新的变化对象

参阅:
* [浅谈JS深拷贝（深克隆）](https://www.jianshu.com/p/f4329eb1bace)
* [JS 中深拷贝的几种实现方法](https://blog.csdn.net/chentony123/article/details/81428803)
* [你不知道的高性能实现深拷贝的方式](https://github.com/KieSun/FE-advance-road/blob/master/wheels/deepClone/index.md)