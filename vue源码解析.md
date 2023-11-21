### **1、变化侦测篇**

#### Object的变化侦测

###### 1.使Object数据变得“可观测”

数据的每次读和写能够被我们看的见，即我们能够知道数据什么时候被读取了或数据什么时候被改写了，我们将其称为数据变的‘可观测’。要将数据变的‘可观测’，我们就要借助前言中提到的`Object.defineProperty`方法了，在本文中，我们就使用这个方法使数据变得“可观测”。

将对象所有属性变为可观测，代码实现如下

```javascript
// 源码位置：src/core/observer/index.js

/**
 * Observer类会通过递归的方式把一个对象的所有属性都转化成可观测对象
 */
export class Observer {
  constructor (value) {
    this.value = value
    // 给value新增一个__ob__属性，值为该value的Observer实例
    // 相当于为value打上标记，表示它已经被转化成响应式了，避免重复操作
    def(value,'__ob__',this)
    if (Array.isArray(value)) {
      // 采用重新写数组原型方法(push,pop,shift,unshift,reverse,splice,sort)来实现数组响应式
    } else {
      this.walk(value)
    }
  }

  walk (obj: Object) {
    const keys = Object.keys(obj)
    for (let i = 0; i < keys.length; i++) {
      defineReactive(obj, keys[i])
    }
  }
}
/**
 * 使一个对象转化成可观测对象
 * @param { Object } obj 对象
 * @param { String } key 对象的key
 * @param { Any } val 对象的某个key的值
 */
function defineReactive (obj,key,val) {
  // 如果只传了obj和key，那么val = obj[key]
  if (arguments.length === 2) {
    val = obj[key]
  }
  // 对象嵌套进行递归遍历
  if(typeof val === 'object'){
      new Observer(val)
  }
  Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    get(){
      console.log(`${key}属性被读取了`);
      return val;
    },
    set(newVal){
      if(val === newVal){
          return
      }
      console.log(`${key}属性被修改了`);
      val = newVal;
    }
  })
};
```

在上面的代码中，我们定义了`observer`类，它用来将一个正常的`object`转换成可观测的`object`。

并且给`value`新增一个`__ob__`属性，值为该`value`的`Observer`实例。这个操作相当于为`value`打上标记，表示它已经被转化成响应式了，避免重复操作。

然后判断数据的类型，只有`object`类型的数据才会调用`walk`将每一个属性转换成`getter/setter`的形式来侦测变化。 最后，在`defineReactive`中当传入的属性值还是一个`object`时使用`new observer（val）`来递归子属性，这样我们就可以把`obj`中的所有属性（包括子属性）都转换成`getter/seter`的形式来侦测变化。 也就是说，只要我们将一个`object`传到`observer`中，那么这个`object`就会变成可观测的、响应式的`object`。

###### 2.依赖收集

谁用到了这个数据，那么当这个数据变化时就通知谁。所谓谁用到了这个数据，其实就是谁获取了这个数据，而可观测的数据被获取时会触发`getter`属性，那么我们就可以在`getter`中收集这个依赖。同样，当这个数据变化时会触发`setter`属性，那么我们就可以在`setter`中通知依赖更新。

我们给每个数据都建一个依赖数组，谁依赖了这个数据我们就把谁放入这个依赖数组中。单单用一个数组来存放依赖的话，功能好像有点欠缺并且代码过于耦合。我们应该将依赖数组的功能扩展一下，更好的做法是我们应该为每一个数据都建立一个依赖管理器，把这个数据所有的依赖都管理起来。

```javascript
// 源码位置：src/core/observer/dep.js
export default class Dep {
  constructor () {
    // 依赖存放数组
    this.subs = []
  }

  addSub (sub) {
    this.subs.push(sub)
  }
  // 删除一个依赖
  removeSub (sub) {
    remove(this.subs, sub)
  }
  // 添加一个依赖
  depend () {
    if (window.target) {
      this.addSub(window.target)
    }
  }
  // 通知所有依赖更新
  notify () {
    const subs = this.subs.slice()
    for (let i = 0, l = subs.length; i < l; i++) {
      subs[i].update()
    }
  }
}

/**
 * Remove an item from an array
 */
export function remove (arr, item) {
  if (arr.length) {
    const index = arr.indexOf(item)
    if (index > -1) {
      return arr.splice(index, 1)
    }
  }
}
```

在上面的依赖管理器`Dep`类中，我们先初始化了一个`subs`数组，用来存放依赖，并且定义了几个实例方法用来对依赖进行添加，删除，通知等操作。

有了依赖管理器后，我们就可以在getter中收集依赖，在setter中通知依赖更新了，代码如下：

```javascript
function defineReactive (obj,key,val) {
  if (arguments.length === 2) {
    val = obj[key]
  }
  if(typeof val === 'object'){
    new Observer(val)
  }
  const dep = new Dep()  //实例化一个依赖管理器，生成一个依赖管理数组dep
  Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    get(){
      dep.depend()    // 在getter中收集依赖
      return val;
    },
    set(newVal){
      if(val === newVal){
          return
      }
      val = newVal;
      dep.notify()   // 在setter中通知依赖更新
    }
  })
}
```

在上述代码中，我们在`getter`中调用了`dep.depend()`方法收集依赖，在`setter`中调用`dep.notify()`方法通知所有依赖更新。

###### 3.依赖到底是谁

在`Vue`中还实现了一个叫做`Watcher`的类，而`Watcher`类的实例就是我们上面所说的那个"谁"。换句话说就是：谁用到了数据，谁就是依赖，我们就为谁创建一个`Watcher`实例。在之后数据变化时，我们不直接去通知依赖更新，而是通知依赖对应的`Watch`实例，由`Watcher`实例去通知真正的视图。

`Watcher`类的具体实现如下：

```javascript
export default class Watcher {
  constructor (vm,expOrFn,cb) {
    this.vm = vm; // vue实例
    this.cb = cb; // watch回调函数
    this.getter = parsePath(expOrFn) // 通过表达式从data对象中获取值
    this.value = this.get()
  }
  get () {
    // 指定全局唯一对象
    window.target = this;
    const vm = this.vm
    // 依赖数据的目的是触发该数据上面的getter，上文我们说过，在getter里会调用dep.depend()收集依赖
    let value = this.getter.call(vm, vm);
    window.target = undefined;
    return value
  }
  // 依赖更新调用方法
  update () {
    const oldValue = this.value
    this.value = this.get()
    this.cb.call(this.vm, this.value, oldValue)
  }
}

/**
 * Parse simple path.
 * 把一个形如'data.a.b.c'的字符串路径所表示的值，从真实的data对象中取出来
 * 例如：
 * data = {a:{b:{c:2}}}
 * parsePath('a.b.c')(data)  // 2
 */
const bailRE = /[^\w.$]/
export function parsePath (path) {
  if (bailRE.test(path)) {
    return
  }
  const segments = path.split('.')
  return function (obj) {
    for (let i = 0; i < segments.length; i++) {
      if (!obj) return
      obj = obj[segments[i]]
    }
    return obj
  }
}
```

谁用到了数据，谁就是依赖，我们就为谁创建一个`Watcher`实例，在创建`Watcher`实例的过程中会自动的把自己添加到这个数据对应的依赖管理器中，以后这个`Watcher`实例就代表这个依赖，当数据变化时，我们就通知`Watcher`实例，由`Watcher`实例再去通知真正的依赖。

`Watcher`类的代码实现逻辑

1. 当实例化`Watcher`类时，会先执行其构造函数；
2. 在构造函数中调用了`this.get()`实例方法；
3. 在`get()`方法中，首先通过`window.target = this`把实例自身赋给了全局的一个唯一对象`window.target`上，然后通过`let value = this.getter.call(vm, vm)`获取一下被依赖的数据，获取被依赖数据的目的是触发该数据上面的`getter`，上文我们说过，在`getter`里会调用`dep.depend()`收集依赖，而在`dep.depend()`中取到挂载`window.target`上的值并将其存入依赖数组中，在`get()`方法最后将`window.target`释放掉。
4. 而当数据变化时，会触发数据的`setter`，在`setter`中调用了`dep.notify()`方法，在`dep.notify()`方法中，遍历所有依赖(即watcher实例)，执行依赖的`update()`方法，也就是`Watcher`类中的`update()`实例方法，在`update()`方法中调用数据变化的更新回调函数，从而更新视图。

简单一句话：`Watcher`先把自己设置到全局唯一的指定位置（`window.target`），然后读取数据。因为读取了数据，所以会触发这个数据的`getter`。接着，在`getter`中就会从全局唯一的那个位置读取当前正在读取数据的`Watcher`，并把这个`watcher`收集到`Dep`中去。收集好之后，当数据发生变化时，会向`Dep`中的每个`Watcher`发送通知。通过这样的方式，`Watcher`可以主动去订阅任意一个数据的变化。

![img](https://vue-js.com/learn-vue/assets/img/3.0b99330d.jpg)

###### 4.不足之处

虽然我们通过`Object.defineProperty`方法实现了对`object`数据的可观测，但是这个方法仅仅只能观测到`object`数据的取值及设置值，当我们向`object`数据里添加一对新的`key/value`或删除一对已有的`key/value`时，它是无法观测到的，导致当我们对`object`数据添加或删除值时，无法通知依赖，无法驱动视图进行响应式更新。

###### 5.总结

其整个流程大致如下：

1. `Data`通过`observer`转换成了`getter/setter`的形式来追踪变化。
2. 当外界通过`Watcher`读取数据时，会触发`getter`从而将`Watcher`添加到依赖中。
3. 当数据发生了变化时，会触发`setter`，从而向`Dep`中的依赖（即Watcher）发送通知。
4. `Watcher`接收到通知后，会向外界发送通知，变化通知到外界后可能会触发视图更新，也有可能触发用户的某个回调函数等。

#### Array的变化侦测

###### 1.使Array型数据可观测

`Vue`中创建了一个数组方法拦截器，它拦截在数组实例与`Array.prototype`之间，在拦截器内重写了操作数组的一些方法，当数组实例使用操作数组方法时，其实使用的是拦截器中重写的方法，而不再使用`Array.prototype`上的原生方法。

```javascript
// 源码位置：/src/core/observer/array.js

const arrayProto = Array.prototype
// 创建一个对象作为拦截器
export const arrayMethods = Object.create(arrayProto)

// 改变数组自身内容的7个方法
const methodsToPatch = [
  'push',
  'pop',
  'shift',
  'unshift',
  'splice',
  'sort',
  'reverse'
]

/**
 * Intercept mutating methods and emit events
 */
methodsToPatch.forEach(function (method) {
  const original = arrayProto[method]      // 缓存原生方法
  Object.defineProperty(arrayMethods, method, {
    enumerable: false,
    configurable: true,
    writable: true,
    value:function mutator(...args){
      const result = original.apply(this, args)
      return result
    }
  })
})
```

重写数组方法后，还要把它挂载到数组实例与`Array.prototype`之间，这样拦截器才能够生效。我们只需把数据的`__proto__`属性设置为拦截器`arrayMethods`即可，源码实现如下：

```javascript
// 源码位置：/src/core/observer/index.js
export class Observer {
  constructor (value) {
    this.value = value
    if (Array.isArray(value)) {
      const augment = hasProto
        ? protoAugment
        : copyAugment
      augment(value, arrayMethods, arrayKeys)
    } else {
      this.walk(value)
    }
  }
}
// 能力检测：判断__proto__是否可用，因为有的浏览器不支持该属性
export const hasProto = '__proto__' in {}

const arrayKeys = Object.getOwnPropertyNames(arrayMethods)

// 将数组原型对象设置为重写后的原型对象
function protoAugment (target, src: Object, keys: any) {
  target.__proto__ = src
}

/**
 * Augment an target Object or Array by defining
 * hidden properties.
 */
/* istanbul ignore next */
function copyAugment (target: Object, src: Object, keys: Array<string>) {
  for (let i = 0, l = keys.length; i < l; i++) {
    const key = keys[i]
    def(target, key, src[key])
  }
}
```

首先判断了浏览器是否支持`__proto__`，如果支持，则调用`protoAugment`函数把`value.__proto__ = arrayMethods`；如果不支持，则调用`copyAugment`函数把拦截器中重写的7个方法循环加入到`value`上。

###### 2.收集依赖

数组的依赖在`getter`中收集，那么在`getter`中到底该如何收集呢？这里有一个需要注意的点，那就是依赖管理器定义在`Observer`类中，而我们需要在`getter`中收集依赖，也就是说我们必须在`getter`中能够访问到`Observer`类中的依赖管理器，才能把依赖存进去。源码是这么做的：

```javascript
function defineReactive (obj,key,val) {
  // 创建一个ob实例，如果存在__ob__属性。说明已经是响应式对象，不需要重复创建
  let childOb = observe(val)
  const dep = new Dep()
  Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    get(){
      if (childOb) {
        childOb.dep.depend()
      }
      return val;
    },
    set(newVal){
      if(val === newVal){
        return
      }
      val = newVal;
      dep.notify()   // 在setter中通知依赖更新
    }
  })
}

/**
 * 尝试为value创建一个0bserver实例，如果创建成功，直接返回新创建的Observer实例。
 * 如果 Value 已经存在一个Observer实例，则直接返回它
 */
export function observe (value, asRootData){
  if (!isObject(value) || value instanceof VNode) {
    return
  }
  let ob
  if (hasOwn(value, '__ob__') && value.__ob__ instanceof Observer) {
    ob = value.__ob__
  } else {
    ob = new Observer(value)
  }
  return ob
}
```

在上面代码中，我们首先通过`observe`函数为被获取的数据`arr`尝试创建一个`Observer`实例，在`observe`函数内部，先判断当前传入的数据上是否有`__ob__`属性，因为在上篇文章中说了，如果数据有`__ob__`属性，表示它已经被转化成响应式的了，如果没有则表示该数据还不是响应式的，那么就调用`new Observer(value)`将其转化成响应式的，并把数据对应的`Observer`实例返回。

而在`defineReactive`函数中，首先获取数据对应的`Observer`实例`childOb`，然后在`getter`中调用`Observer`实例上依赖管理器，从而将依赖收集起来。

###### 3.如何通知依赖

我们应该在拦截器里通知依赖，要想通知依赖，首先要能访问到依赖。要访问到依赖也不难，因为我们只要能访问到被转化成响应式的数据`value`即可，因为`vaule`上的`__ob__`就是其对应的`Observer`类实例，有了`Observer`类实例我们就能访问到它上面的依赖管理器，然后只需调用依赖管理器的`dep.notify()`方法，让它去通知依赖更新即可。源码如下：

```javascript
/**
 * Intercept mutating methods and emit events
 */
methodsToPatch.forEach(function (method) {
  const original = arrayProto[method]
  def(arrayMethods, method, function mutator (...args) {
    const result = original.apply(this, args)
    // ob实例
    const ob = this.__ob__
    // notify change
    ob.dep.notify()
    return result
  })
})
```

上面代码中，由于我们的拦截器是挂载到数组数据的原型上的，所以拦截器中的`this`就是数据`value`，拿到`value`上的`Observer`类实例，从而你就可以调用`Observer`类实例上面依赖管理器的`dep.notify()`方法，以达到通知依赖的目的。

###### 4.深度侦测

`Array`型数据的变化侦测都仅仅说的是数组自身变化的侦测，比如给数组新增一个元素或删除数组中一个元素，而在`Vue`中，不论是`Object`型数据还是`Array`型数据所实现的数据变化侦测都是深度侦测，所谓深度侦测就是不但要侦测数据自身的变化，还要侦测数据中所有子数据的变化。

```javascript
export class Observer {
  value: any;
  dep: Dep;

  constructor (value: any) {
    this.value = value
    this.dep = new Dep()
    def(value, '__ob__', this)
    if (Array.isArray(value)) {
      const augment = hasProto
        ? protoAugment
        : copyAugment
      augment(value, arrayMethods, arrayKeys)
      this.observeArray(value)   // 将数组中的所有元素都转化为可被侦测的响应式
    } else {
      this.walk(value)
    }
  }

  /**
   * Observe a list of Array items.
   */
  observeArray (items: Array<any>) {
    for (let i = 0, l = items.length; i < l; i++) {
      observe(items[i])
    }
  }
}

export function observe (value, asRootData){
  if (!isObject(value) || value instanceof VNode) {
    return
  }
  let ob
  if (hasOwn(value, '__ob__') && value.__ob__ instanceof Observer) {
    ob = value.__ob__
  } else {
    ob = new Observer(value)
  }
  return ob
}
```

对于数组中已有的元素我们已经可以将其全部转化成可侦测的响应式数据了，但是如果向数组里新增一个元素的话，我们也需要将新增的这个元素转化成可侦测的响应式数据。

这个实现起来也很容易，我们只需拿到新增的这个元素，然后调用`observe`函数将其转化即可。我们知道，可以向数组内新增元素的方法有3个，分别是：`push`、`unshift`、`splice`。我们只需对这3中方法分别处理，拿到新增的元素，再将其转化即可。源码如下：

```java
/**
 * Intercept mutating methods and emit events
 */
methodsToPatch.forEach(function (method) {
  // cache original method
  const original = arrayProto[method]
  def(arrayMethods, method, function mutator (...args) {
    const result = original.apply(this, args)
    const ob = this.__ob__
    let inserted
    switch (method) {
      case 'push':
      case 'unshift':
        inserted = args   // 如果是push或unshift方法，那么传入参数就是新增的元素
        break
      case 'splice':
        inserted = args.slice(2) // 如果是splice方法，那么传入参数列表中下标为2的就是新增的元素
        break
    }
    if (inserted) ob.observeArray(inserted) // 调用observe函数将新增的元素转化成响应式
    // notify change
    ob.dep.notify()
    return result
  })
})
```

###### 5.不足之处

对于数组变化侦测是通过拦截器实现的，也就是说只要是通过数组原型上的方法对数组进行操作就都可以侦测到，但是别忘了，我们在日常开发中，还可以通过数组的下标来操作数据，如下：

```javascript
let arr = [1,2,3]
arr[0] = 5;       // 通过数组下标修改数组中的数据
arr.length = 0    // 通过修改数组长度清空数组
```

而使用上述例子中的操作方式来修改数组是无法侦测到的。 同样，`Vue`也注意到了这个问题， 为了解决这一问题，`Vue`增加了两个全局API:`Vue.set`和`Vue.delete`。

### 2、虚拟DOM

#### Vue中的虚拟DOM

##### Vue中的DOM-Diff

我们知道，`Vue`是数据驱动视图的，数据发生变化视图就要随之更新，在更新视图的时候难免要操作`DOM`,而操作真实`DOM`又是非常耗费性能的，我们可以用`JS`的计算性能来换取操作`DOM`所消耗的性能。我们不要盲目的去更新视图，而是通过对比数据变化前后的状态，计算出视图中哪些地方需要更新，只更新需要更新的地方，而不需要更新的地方则不需关心，这样我们就可以尽可能少的操作`DOM`了。

###### VNode类

```javascript
// 源码位置：src/core/vdom/vnode.js

export default class VNode {
  constructor (
    tag?: string,
    data?: VNodeData,
    children?: ?Array<VNode>,
    text?: string,
    elm?: Node,
    context?: Component,
    componentOptions?: VNodeComponentOptions,
    asyncFactory?: Function
  ) {
    this.tag = tag                                /*当前节点的标签名*/
    this.data = data        /*当前节点对应的对象，包含了具体的一些数据信息，是一个VNodeData类型，可以参考VNodeData类型中的数据信息*/
    this.children = children  /*当前节点的子节点，是一个数组*/
    this.text = text     /*当前节点的文本*/
    this.elm = elm       /*当前虚拟节点对应的真实dom节点*/
    this.ns = undefined            /*当前节点的名字空间*/
    this.context = context          /*当前组件节点对应的Vue实例*/
    this.fnContext = undefined       /*函数式组件对应的Vue实例*/
    this.fnOptions = undefined
    this.fnScopeId = undefined
    this.key = data && data.key           /*节点的key属性，被当作节点的标志，用以优化*/
    this.componentOptions = componentOptions   /*组件的option选项*/
    this.componentInstance = undefined       /*当前节点对应的组件的实例*/
    this.parent = undefined           /*当前节点的父节点*/
    this.raw = false         /*简而言之就是是否为原生HTML或只是普通文本，innerHTML的时候为true，textContent的时候为false*/
    this.isStatic = false         /*静态节点标志*/
    this.isRootInsert = true      /*是否作为跟节点插入*/
    this.isComment = false             /*是否为注释节点*/
    this.isCloned = false           /*是否为克隆节点*/
    this.isOnce = false                /*是否有v-once指令*/
    this.asyncFactory = asyncFactory
    this.asyncMeta = undefined
    this.isAsyncPlaceholder = false
  }

  get child (): Component | void {
    return this.componentInstance
  }
}
```

###### VNode的类型

- 注释节点
- 文本节点
- 元素节点
- 组件节点
- 函数式组件节点
- 克隆节点

###### 注释节点

```javascript
// 创建注释节点
export const createEmptyVNode = (text: string = '') => {
  const node = new VNode()
  node.text = text
  node.isComment = true
  return node
}
```

从上面代码中可以看到，描述一个注释节点只需两个属性，分别是：`text`和`isComment`。其中`text`属性表示具体的注释信息，`isComment`是一个标志，用来标识一个节点是否是注释节点。

###### 文本节点

文本节点描述起来比注释节点更简单，因为它只需要一个属性，那就是`text`属性，用来表示具体的文本信息。

```javascript
// 创建文本节点
export function createTextVNode (val: string | number) {
  return new VNode(undefined, undefined, undefined, String(val))
}
```

###### 元素节点

相比之下，元素节点更贴近于我们通常看到的真实`DOM`节点，它有描述节点标签名词的`tag`属性，描述节点属性如`class`、`attributes`等的`data`属性，有描述包含的子节点信息的`children`属性等。

```javascript
// 真实DOM节点
<div id='a'><span>难凉热血</span></div>

// VNode节点
{
  tag:'div',
  data:{},
  children:[
    {
      tag:'span',
      text:'难凉热血'
    }
  ]
}
```

###### 组件节点

组件节点除了有元素节点具有的属性之外，它还有两个特有的属性：

- componentOptions :组件的option选项，如组件的`props`等
- componentInstance :当前组件节点对应的`Vue`实例

###### 函数式组件节点

函数式组件节点相较于组件节点，它又有两个特有的属性：

- fnContext:函数式组件对应的Vue实例
- fnOptions: 组件的option选项

###### 克隆节点

克隆节点就是把一个已经存在的节点复制一份出来，它主要是为了做模板编译优化时使用。

```	javascript
// 创建克隆节点
export function cloneVNode (vnode: VNode): VNode {
  const cloned = new VNode(
    vnode.tag,
    vnode.data,
    vnode.children,
    vnode.text,
    vnode.elm,
    vnode.context,
    vnode.componentOptions,
    vnode.asyncFactory
  )
  cloned.ns = vnode.ns
  cloned.isStatic = vnode.isStatic
  cloned.key = vnode.key
  cloned.isComment = vnode.isComment
  cloned.fnContext = vnode.fnContext
  cloned.fnOptions = vnode.fnOptions
  cloned.fnScopeId = vnode.fnScopeId
  cloned.asyncMeta = vnode.asyncMeta
  cloned.isCloned = true
  return cloned
}
```

##### Vue中的虚拟DOM的Diff

`VNode`最大的用途就是在数据变化前后生成真实`DOM`对应的虚拟`DOM`节点，然后就可以对比新旧两份`VNode`，找出差异所在，然后更新有差异的`DOM`节点，最终达到以最少操作真实`DOM`更新视图的目的。

整个`patch`无非就是干三件事：

- 创建节点：新的`VNode`中有而旧的`oldVNode`中没有，就在旧的`oldVNode`中创建。
- 删除节点：新的`VNode`中没有而旧的`oldVNode`中有，就从旧的`oldVNode`中删除。
- 更新节点：新的`VNode`和旧的`oldVNode`中都有，就以新的`VNode`为准，更新旧的`oldVNode`。

###### 创建节点

```javascript
// 源码位置: /src/core/vdom/patch.js
function createElm (vnode, parentElm, refElm) {
    const data = vnode.data
    const children = vnode.children
    const tag = vnode.tag
    if (isDef(tag)) {
      	vnode.elm = nodeOps.createElement(tag, vnode)   // 创建元素节点
        createChildren(vnode, children, insertedVnodeQueue) // 创建元素节点的子节点
        insert(parentElm, vnode.elm, refElm)       // 插入到DOM中
    } else if (isTrue(vnode.isComment)) {
      vnode.elm = nodeOps.createComment(vnode.text)  // 创建注释节点
      insert(parentElm, vnode.elm, refElm)           // 插入到DOM中
    } else {
      vnode.elm = nodeOps.createTextNode(vnode.text)  // 创建文本节点
      insert(parentElm, vnode.elm, refElm)           // 插入到DOM中
    }
  }
```

- 判断是否为元素节点只需判断该`VNode`节点是否有`tag`标签即可。如果有`tag`属性即认为是元素节点，则调用`createElement`方法创建元素节点，通常元素节点还会有子节点，那就递归遍历创建所有子节点，将所有子节点创建好之后`insert`插入到当前元素节点里面，最后把当前元素节点插入到`DOM`中。
- 判断是否为注释节点，只需判断`VNode`的`isComment`属性是否为`true`即可，若为`true`则为注释节点，则调用`createComment`方法创建注释节点，再插入到`DOM`中。
- 如果既不是元素节点，也不是注释节点，那就认为是文本节点，则调用`createTextNode`方法创建文本节点，再插入到`DOM`中。

###### 删除节点

如果某些节点再新的`VNode`中没有而在旧的`oldVNode`中有，那么就需要把这些节点从旧的`oldVNode`中删除。删除节点非常简单，只需在要删除节点的父元素上调用`removeChild`方法即可。源码如下：

```javascript
function removeNode (el) {
    const parent = nodeOps.parentNode(el)  // 获取父节点
    if (isDef(parent)) {
      nodeOps.removeChild(parent, el)  // 调用父节点的removeChild方法
    }
  }
```

###### 更新节点

```javascript
1、当修改数据时，触发set属性Dep.notify()方法通知对应的订阅者，然后调用patch(oldNode,newNode)方法进行比对。
2、调用isSameVNode方法判断新旧节点是否是相同类型标签，如果是不同类型标签，则直接替换。相同类型标签则进行下一步比较。
3、调用patchNode方法判断新旧节点是否相等，如果相等直接返回，不等则进行下一步比较。
4、判断新旧节点都有文本节点，用新文本替换旧文本。
   旧节点没有子节点，新节点有子节点。需要增加新的子节点。
   旧节点有子节点，新节点没有有子节点。需要删除旧的子节点。
   新旧都有子节点，需要调用updateChildren方法进行比较。
5、updateChildren采用首尾指针方法对比方法，只对比同级节点，提高对比效率。
   每个节点都有start点和end点，用旧节点start点和end点去比对新节点的start或end点，每次比较成功后，start点和end点都会向中间靠拢，当新旧节点中有一个start点跑到end点右侧后，就终止比较。
   如果上述情况匹配不到，则需要用旧节点key值去比对新节点key值，如果key值相同则复用，并将旧节点移动到新节点位置。
```

### 3、模版编译

所谓渲染流程，就是把用户写的类似于原生`HTML`的模板经过一系列处理最终反应到视图中称之为整个渲染流程。

![img](https://vue-js.com/learn-vue/assets/img/1.f0570125.png)

#### 具体流程

将一堆字符串模板解析成抽象语法树`AST`后，我们就可以对其进行各种操作处理了，处理完后用处理后的`AST`来生成`render`函数。其具体流程可大致分为三个阶段：

1. 模板解析阶段：将一堆模板字符串用正则等方式解析成抽象语法树`AST`；
2. 优化阶段：遍历`AST`，找出其中的静态节点，并打上标记；
3. 代码生成阶段：将`AST`转换成渲染函数；

```javascript
export const createCompiler = createCompilerCreator(function baseCompile (
  template: string,
  options: CompilerOptions
): CompiledResult {
  // 模板解析阶段：用正则等方式解析 template 模板中的指令、class、style等数据，形成AST
  const ast = parse(template.trim(), options)
  if (options.optimize !== false) {
    // 优化阶段：遍历AST，找出其中的静态节点，并打上标记；
    optimize(ast, options)
  }
  // 代码生成阶段：将AST转换成渲染函数；
  const code = generate(ast, options)
  return {
    ast,
    render: code.render,
    staticRenderFns: code.staticRenderFns
  }
})
```

#### 模板解析阶段(整体运行流程)

模板解析其实就是根据被解析内容的特点使用正则等方式将有效信息解析提取出来，根据解析内容的不同分为HTML解析器，文本解析器和过滤器解析器。而文本信息与过滤器信息又存在于HTML标签中，所以在解析器主线函数`parse`中先调用HTML解析器`parseHTML` 函数对模板字符串进行解析，如果在解析过程中遇到文本或过滤器信息则再调用相应的解析器进行解析，最终完成对整个模板字符串的解析。

#### 模板解析阶段(HTML解析器)

```javascript
// 代码位置：/src/complier/parser/index.js

/**
 * Convert HTML string to AST.
 * 将HTML模板字符串转化为AST
 */
export function parse(template, options) {
   // ...
  parseHTML(template, {
    warn,
    expectHTML: options.expectHTML,
    isUnaryTag: options.isUnaryTag, // 是否为闭合标签
    canBeLeftOpenTag: options.canBeLeftOpenTag,
    shouldDecodeNewlines: options.shouldDecodeNewlines,
    shouldDecodeNewlinesForHref: options.shouldDecodeNewlinesForHref,
    shouldKeepComment: options.comments,
    // 当解析到开始标签时，调用该函数
    start (tag, attrs, unary) {

    },
    // 当解析到结束标签时，调用该函数
    end () {

    },
    // 当解析到文本时，调用该函数
    chars (text) {

    },
    // 当解析到注释时，调用该函数
    comment (text) {

    }
  })
  return root
}
```

当解析到开始标签时调用`start`函数生成元素类型的`AST`节点，代码如下；

```javascript
// 当解析到标签的开始位置时，触发start
start (tag, attrs, unary) {
	let element = createASTElement(tag, attrs, currentParent)
}

export function createASTElement (tag,attrs,parent) {
  return {
    type: 1,
    tag,
    attrsList: attrs,
    attrsMap: makeAttrsMap(attrs),
    parent,
    children: []
  }
}
```

当解析到结束标签时调用`end`函数；

当解析到文本时调用`chars`函数生成文本类型的`AST`节点；

```javascript
// 当解析到标签的文本时，触发chars
chars (text) {
	if(text是带变量的动态文本){
    let element = {
      type: 2,
      expression: res.expression,
      tokens: res.tokens,
      text
    }
  } else {
    let element = {
      type: 3,
      text
    }
  }
}
```

当解析到注释时调用`comment`函数生成注释类型的`AST`节点；

```javascript
// 当解析到标签的注释时，触发comment
comment (text: string) {
  let element = {
    type: 3,
    text,
    isComment: true
  }
}
```

##### 如何解析不同的内容

要从模板字符串中解析出不同的内容，那首先要知道模板字符串中都会包含哪些内容。那么通常我们所写的模板字符串中都会包含哪些内容呢？经过整理，通常模板内会包含如下内容：

- 文本，例如“难凉热血”
- HTML注释，例如<!-- 我是注释 -->
- 条件注释，例如<!-- [if !IE]> -->我是注释<!--< ![endif] -->
- DOCTYPE，例如<!DOCTYPE html>
- 开始标签，例如<div>
- 结束标签，例如</div>

###### 解析HTML注释

解析注释比较简单，我们知道`HTML`注释是以`<!--`开头，以`-->`结尾，这两者中间的内容就是注释内容，那么我们只需用正则判断待解析的模板字符串`html`是否以`<!--`开头，若是，那就继续向后寻找`-->`，如果找到了，OK，注释就被解析出来了。

```javascript
const comment = /^<!\--/
if (comment.test(html)) {
  // 若为注释，则继续查找是否存在'-->'
  const commentEnd = html.indexOf('-->')

  if (commentEnd >= 0) {
    // 若存在 '-->',继续判断options中是否保留注释
    if (options.shouldKeepComment) {
      // 若保留注释，则把注释截取出来传给options.comment，创建注释类型的AST节点
      options.comment(html.substring(4, commentEnd))
    }
    // 若不保留注释，则将游标移动到'-->'之后，继续向后解析
    advance(commentEnd + 3)
    continue
  }
}
```

###### 解析条件注释

解析条件注释也比较简单，其原理跟解析注释相同，都是先用正则判断是否是以条件注释特有的开头标识开始，然后寻找其特有的结束标识，若找到，则说明是条件注释，将其截取出来即可，由于条件注释不存在于真正的`DOM`树中，所以不需要调用钩子函数创建`AST`节点。代码如下：

```javascript
// 解析是否是条件注释
const conditionalComment = /^<!\[/
if (conditionalComment.test(html)) {
  // 若为条件注释，则继续查找是否存在']>'
  const conditionalEnd = html.indexOf(']>')

  if (conditionalEnd >= 0) {
    // 若存在 ']>',则从原本的html字符串中把条件注释截掉，
    // 把剩下的内容重新赋给html，继续向后匹配
    advance(conditionalEnd + 2)
    continue
  }
}
```

###### 解析DOCTYPE

解析`DOCTYPE`的原理同解析条件注释完全相同，此处不再赘述，代码如下：

```javascript
const doctype = /^<!DOCTYPE [^>]+>/i
// 解析是否是DOCTYPE
const doctypeMatch = html.match(doctype)
if (doctypeMatch) {
  advance(doctypeMatch[0].length)
  continue
}
```

###### 解析开始标签

首先使用开始标签的正则去匹配模板字符串，看模板字符串是否具有开始标签的特征，如下：

```javascript
/**
 * 匹配开始标签的正则
 */
const ncname = '[a-zA-Z_][\\w\\-\\.]*'
const qnameCapture = `((?:${ncname}\\:)?${ncname})`
const startTagOpen = new RegExp(`^<${qnameCapture}`)

const start = html.match(startTagOpen)
if (start) {
  const match = {
    tagName: start[1],
    attrs: [],
    start: index
  }
}

// 以开始标签开始的模板：
'<div></div>'.match(startTagOpen)  => ['<div','div',index:0,input:'<div></div>']
// 以结束标签开始的模板：
'</div><div></div>'.match(startTagOpen) => null
// 以文本开始的模板：
'我是文本</p>'.match(startTagOpen) => null
```

###### 解析结束标签

结束标签的解析要比解析开始标签容易多了，因为它不需要解析什么属性，只需要判断剩下的模板字符串是否符合结束标签的特征，如果是，就将结束标签名提取出来，再调用4个钩子函数中的`end`函数就好了。

首先判断剩余的模板字符串是否符合结束标签的特征，如下：

```javascript
const ncname = '[a-zA-Z_][\\w\\-\\.]*'
const qnameCapture = `((?:${ncname}\\:)?${ncname})`
const endTag = new RegExp(`^<\\/${qnameCapture}[^>]*>`)
const endTagMatch = html.match(endTag)

'</div>'.match(endTag)  // ["</div>", "div", index: 0, input: "</div>", groups: undefined]
'<div>'.match(endTag)  // null
```

###### 解析文本

解析文本也比较容易，在解析模板字符串之前，我们先查找一下第一个`<`出现在什么位置，如果第一个`<`在第一个位置，那么说明模板字符串是以其它5种类型开始的；如果第一个`<`不在第一个位置而在模板字符串中间某个位置，那么说明模板字符串是以文本开头的，那么从开头到第一个`<`出现的位置就都是文本内容了；如果在整个模板字符串里没有找到`<`，那说明整个模板字符串都是文本。这就是解析思路，接下来我们对照源码来了解一下实际的解析过程，源码如下：

```javascript
let textEnd = html.indexOf('<')
// '<' 在第一个位置，为其余5种类型
if (textEnd === 0) {
    // ...
}
// '<' 不在第一个位置，文本开头
if (textEnd >= 0) {
    // 如果html字符串不是以'<'开头,说明'<'前面的都是纯文本，无需处理
    // 那就把'<'以后的内容拿出来赋给rest
    rest = html.slice(textEnd)
    while (
        !endTag.test(rest) &&
        !startTagOpen.test(rest) &&
        !comment.test(rest) &&
        !conditionalComment.test(rest)
    ) {
        // < in plain text, be forgiving and treat it as text
        /**
           * 用'<'以后的内容rest去匹配endTag、startTagOpen、comment、conditionalComment
           * 如果都匹配不上，表示'<'是属于文本本身的内容
           */
        // 在'<'之后查找是否还有'<'
        next = rest.indexOf('<', 1)
        // 如果没有了，表示'<'后面也是文本
        if (next < 0) break
        // 如果还有，表示'<'是文本中的一个字符
        textEnd += next
        // 那就把next之后的内容截出来继续下一轮循环匹配
        rest = html.slice(textEnd)
    }
    // '<'是结束标签的开始 ,说明从开始到'<'都是文本，截取出来
    text = html.substring(0, textEnd)
    advance(textEnd)
}
// 整个模板字符串里没有找到`<`,说明整个模板字符串都是文本
if (textEnd < 0) {
    text = html
    html = ''
}
// 把截取出来的text转化成textAST
if (options.chars && text) {
    options.chars(text)
}
```

##### 如何保证AST节点层级关系

`Vue`在`HTML`解析器的开头定义了一个栈`stack`，这个栈的作用就是用来维护`AST`节点层级的，那么它是怎么维护的呢？通过前文我们知道，`HTML`解析器在从前向后解析模板字符串时，每当遇到开始标签时就会调用`start`钩子函数，那么在`start`钩子函数内部我们可以将解析得到的开始标签推入栈中，而每当遇到结束标签时就会调用`end`钩子函数，那么我们也可以在`end`钩子函数内部将解析得到的结束标签所对应的开始标签从栈中弹出。

```html
<div><p><span></span></p></div>
```



![img](https://vue-js.com/learn-vue/assets/img/7.6ca1dbf0.png)

这样我们就找到了当前被构建节点的父节点。这只是栈的一个用途，它还有另外一个用途，我们再看如下模板字符串：

```html
<div><p><span></p></div>
```

按照上面的流程解析这个模板字符串时，当解析到结束标签`</p>`时，此时栈顶的标签应该是`p`才对，而现在是`span`，那么就说明`span`标签没有被正确闭合，此时控制台就会抛出警告：‘tag has no matching end tag.’相信这个警告你一定不会陌生。这就是栈的第二个用途： 检测模板字符串中是否有未正确闭合的标签。

#### 模板解析阶段(文本解析器)

```javascript
// 当解析到标签的文本时，触发chars
chars (text) {
  if(res = parseText(text)){
       let element = {
           type: 2,
           expression: res.expression,
           tokens: res.tokens,
           text
       }
    } else {
       let element = {
           type: 3,
           text
       }
    }
}
```

文本解析器内部就干了三件事：

- 判断传入的文本是否包含变量
- 构造expression
- 构造tokens



#### 优化阶段

优化阶段其实就干了两件事：

1. 在`AST`中找出所有静态节点并打上标记；
2. 在`AST`中找出所有静态根节点并打上标记；

```javascript
export function optimize (root: ?ASTElement, options: CompilerOptions) {
  if (!root) return
  isStaticKey = genStaticKeysCached(options.staticKeys || '')
  isPlatformReservedTag = options.isReservedTag || no
  // 标记静态节点
  markStatic(root)
  // 标记静态根节点
  markStaticRoots(root, false)
}
```

###### 标记静态节点

判断是否为静态节点：

```javascript
function isStatic (node: ASTNode): boolean {
  if (node.type === 2) { // 包含变量的动态文本节点
    return false
  }
  if (node.type === 3) { // 不包含变量的纯文本节点
    return true
  }
  return !!(node.pre || (
    !node.hasBindings && // no dynamic bindings
    !node.if && !node.for && // not v-if or v-for or v-else
    !isBuiltInTag(node.tag) && // not a built-in
    isPlatformReservedTag(node.tag) && // not a component
    !isDirectChildOfTemplateFor(node) &&
    Object.keys(node).every(isStaticKey)
  ))
}
```

如果元素节点是静态节点，那就必须满足以下几点要求：

- 如果节点使用了`v-pre`指令，那就断定它是静态节点；
- 如果节点没有使用`v-pre`指令，那它要成为静态节点必须满足：
  - 不能使用动态绑定语法，即标签上不能有`v-`、`@`、`:`开头的属性；
  - 不能使用`v-if`、`v-else`、`v-for`指令；
  - 不能是内置组件，即标签名不能是`slot`和`component`；
  - 标签名必须是平台保留标签，即不能是组件；
  - 当前节点的父节点不能是带有 `v-for` 的 `template` 标签；
  - 节点的所有属性的 `key` 都必须是静态节点才有的 `key`，注：静态节点的`key`是有限的，它只能是`type`,`tag`,`attrsList`,`attrsMap`,`plain`,`parent`,`children`,`attrs`之一；

###### 标记静态根节点

判断是否为静态根节点：

```javascript
// 为了使节点有资格作为静态根节点，它应具有不只是静态文本的子节点。 否则，优化的成本将超过收益，最好始终将其更新。
if (node.static && node.children.length && !(
    node.children.length === 1 &&
    node.children[0].type === 3
)) {
    node.staticRoot = true
    return
} else {
    node.staticRoot = false
}
```

一个节点要想成为静态根节点，它必须满足以下要求：

- 节点本身必须是静态节点；
- 必须拥有子节点 `children`；
- 子节点不能只是只有一个文本节点；



### 4、生命周期

下图是`Vue`官网给出的`Vue`实例的生命周期流程图，如下：

<img src="https://vue-js.com/learn-vue/assets/img/1.6e1e57be.jpg" alt="img" style="zoom: 67%;" />

`Vue`实例的生命周期大致可分为4个阶段：

- 初始化阶段：为`Vue`实例上初始化一些属性，事件以及响应式数据；
- 模板编译阶段：将模板编译成渲染函数；
- 挂载阶段：将实例挂载到指定的`DOM`上，即将模板渲染到真实`DOM`中；
- 销毁阶段：将实例自身从父组件中删除，并取消依赖追踪及事件监听器；

#### 初始化阶段(new Vue)

合并配置，调用一些初始化函数，触发生命周期钩子函数，调用`$mount`开启下一个阶段。

```javascript
function Vue (options) {
  if (process.env.NODE_ENV !== 'production' &&
    !(this instanceof Vue)
  ) {
    warn('Vue is a constructor and should be called with the `new` keyword')
  }
  this._init(options) // 执行的是Vue原型上
}

// Vue类的原型上绑定_init方法
export function initMixin (Vue) {
  Vue.prototype._init = function (options) {
    const vm = this
    // 用户传递的options选项与当前构造函数的options属性及其父级构造函数的options属性进行合并
    vm.$options = mergeOptions(
        resolveConstructorOptions(vm.constructor),
        options || {},
        vm
    )
    vm._self = vm
    initLifecycle(vm)       // 初始化生命周期
    initEvents(vm)        // 初始化事件
    initRender(vm)         // 初始化渲染
    callHook(vm, 'beforeCreate')  // 调用生命周期钩子函数
    initInjections(vm)   //初始化injections
    initState(vm)    // 初始化props,methods,data,computed,watch
    initProvide(vm) // 初始化 provide
    callHook(vm, 'created')  // 调用生命周期钩子函数

    if (vm.$options.el) {
      vm.$mount(vm.$options.el)
    }
  }
}
```

##### 合并属性

它实际上就是把 `resolveConstructorOptions(vm.constructor)` 的返回值和 `options` 做合并，`resolveConstructorOptions` 的实现先不考虑，可简单理解为返回 `vm.constructor.options`，相当于 `Vue.options`。

```javascript
vm.$options = mergeOptions(
    resolveConstructorOptions(vm.constructor),
    options || {},
    vm
)

export function initGlobalAPI (Vue: GlobalAPI) {
  // 获取Vue默认配置
  Vue.options = Object.create(null)
  ASSET_TYPES.forEach(type => {
    Vue.options[type + 's'] = Object.create(null)
  })

  // 置组件扩展到 Vue.options.components 上，Vue 的内置组件目前 有<keep-alive>、<transition> 和<transition-group> 组件，这也就是为什么我们在其它组件中使用这些组件不需要注册的原因。
  extend(Vue.options.components, builtInComponents)
  // ...
}
```

`mergeOptions`函数的 主要功能是把 `parent` 和 `child` 这两个对象根据一些合并策略，合并成一个新对象并返回。首先递归把 `extends` 和 `mixins` 合并到 `parent` 上，然后创建一个空对象`options`，遍历 `parent`，把`parent`中的每一项通过调用 `mergeField`函数合并到空对象`options`里，接着再遍历 `child`，把存在于`child`里但又不在 `parent`中 的属性继续调用 `mergeField`函数合并到空对象`options`里，

```javascript
/**
 * Merge two option objects into a new one.
 * Core utility used in both instantiation and inheritance.
 */
export function mergeOptions (
  parent: Object,
  child: Object,
  vm?: Component
): Object {

  if (typeof child === 'function') {
    child = child.options
  }
  const extendsFrom = child.extends
  if (extendsFrom) {
    parent = mergeOptions(parent, extendsFrom, vm)
  }
  if (child.mixins) {
    for (let i = 0, l = child.mixins.length; i < l; i++) {
      parent = mergeOptions(parent, child.mixins[i], vm)
    }
  }
  const options = {}
  let key
  for (key in parent) {
    mergeField(key)
  }
  for (key in child) {
    if (!hasOwn(parent, key)) {
      mergeField(key)
    }
  }
  function mergeField (key) {
    const strat = strats[key] || defaultStrat
    options[key] = strat(parent[key], child[key], vm, key)
  }
  return options
}
```

生命周期钩子函数的合并策略如下：

```javascript

/**
 * Hooks and props are merged as arrays.
 */
// 如果 childVal不存在，就返回 parentVal；否则再判断是否存在 parentVal，如果存在就把 childVal 添加到 parentVal 后返回新数组；否则返回 childVal 的数组。所以回到 mergeOptions 函数，一旦 parent 和 child 都定义了相同的钩子函数，那么它们会把 2 个钩子函数合并成一个数组。合并为数组是为了后面Vue.mixin用户自定义合并。
function mergeHook (parentVal,childVal):  {
  return childVal
    ? parentVal
      ? parentVal.concat(childVal)
      : Array.isArray(childVal)
        ? childVal
        : [childVal]
    : parentVal
}

LIFECYCLE_HOOKS.forEach(hook => {
  strats[hook] = mergeHook
})

export const LIFECYCLE_HOOKS = [
  'beforeCreate',
  'created',
  'beforeMount',
  'mounted',
  'beforeUpdate',
  'updated',
  'beforeDestroy',
  'destroyed',
  'activated',
  'deactivated',
  'errorCaptured'
]
```

#### callHook函数如何触发钩子函数

`callHook`函数逻辑非常简单。首先从实例的`$options`中获取到需要触发的钩子名称所对应的钩子函数数组`handlers`，我们说过，每个生命周期钩子名称都对应了一个钩子函数数组。然后遍历该数组，将数组中的每个钩子函数都执行一遍。

```javascript
export function callHook (vm: Component, hook: string) {
  const handlers = vm.$options[hook]
  if (handlers) {
    for (let i = 0, j = handlers.length; i < j; i++) {
      try {
        handlers[i].call(vm)
      } catch (e) {
        handleError(e, vm, `${hook} hook`)
      }
    }
  }
}
```

#### 初始化阶段(initLifecycle)【生命周期】

主要是给`Vue`实例上挂载了一些属性并设置了默认值。

```javascript
export function initLifecycle (vm: Component) {
  const options = vm.$options

  // 如果当前组件不是抽象组件并且存在父级，那么就通过while循环来向上循环，如果当前组件的父级是抽象组件并且也存在父级，那就继续向上查找当前组件父级的父级，直到找到第一个不是抽象类型的父级时，将其赋值vm.$parent，同时把该实例自身添加进找到的父级的$children属性中。这样就确保了在子组件的$parent属性上能访问到父组件实例，在父组件的$children属性上也能访问子组件的实例。
  let parent = options.parent
  if (parent && !options.abstract) {
    while (parent.$options.abstract && parent.$parent) {
      parent = parent.$parent
    }
    parent.$children.push(vm)
  }

  vm.$parent = parent
  // 首先会判断如果当前实例存在父级，那么当前实例的根实例$root属性就是其父级的根实例$root属性，如果不存在，那么根实例$root属性就是它自己。
  vm.$root = parent ? parent.$root : vm

  vm.$children = []
  vm.$refs = {}

  vm._watcher = null
  vm._inactive = null
  vm._directInactive = false
  vm._isMounted = false
  vm._isDestroyed = false
  vm._isBeingDestroyed = false
}
```

#### 初始化阶段(initEvents)【事件绑定】

##### 解析事件

```javascript
export const onRE = /^@|^v-on:/
export const dirRE = /^v-|^@|^:/

function processAttrs (el) {
  const list = el.attrsList
  let i, l, name, value, modifiers
  for (i = 0, l = list.length; i < l; i++) {
    name  = list[i].name
    value = list[i].value
    if (dirRE.test(name)) {
      // 解析修饰符
      modifiers = parseModifiers(name)
      if (modifiers) {
        name = name.replace(modifierRE, '')
      }
      if (onRE.test(name)) { // v-on
        name = name.replace(onRE, '')
        addHandler(el, name, value, modifiers, false, warn)
      }
    }
  }
}
```

首先根据 `modifier` 修饰符对事件名 `name` 做处理，接着根据 `modifier.native` 判断事件是一个浏览器原生事件还是自定义事件，分别对应 `el.nativeEvents` 和 `el.events`，最后按照 `name` 对事件做归类，并把回调函数的字符串保留到对应的事件中。

```javascript
export function addHandler (el,name,value,modifiers) {
  modifiers = modifiers || emptyObject

  // check capture modifier 判断是否有capture修饰符
  if (modifiers.capture) {
    delete modifiers.capture
    name = '!' + name // 给事件名前加'!'用以标记capture修饰符
  }
  // 判断是否有once修饰符
  if (modifiers.once) {
    delete modifiers.once
    name = '~' + name // 给事件名前加'~'用以标记once修饰符
  }
  // 判断是否有passive修饰符
  if (modifiers.passive) {
    delete modifiers.passive
    name = '&' + name // 给事件名前加'&'用以标记passive修饰符
  }

  let events
  if (modifiers.native) {
    delete modifiers.native
    events = el.nativeEvents || (el.nativeEvents = {})
  } else {
    events = el.events || (el.events = {})
  }

  const newHandler: any = {
    value: value.trim()
  }
  if (modifiers !== emptyObject) {
    newHandler.modifiers = modifiers
  }

  const handlers = events[name]
  if (Array.isArray(handlers)) {
    handlers.push(newHandler)
  } else if (handlers) {
    events[name] = [handlers, newHandler]
  } else {
    events[name] = newHandler
  }

  el.plain = false
}
```

然后在模板编译的代码生成阶段，会在 `genData` 函数中根据 `AST` 元素节点上的 `events` 和 `nativeEvents` 生成`_c(tagName,data,children)`函数中所需要的 `data` 数据。

```javascript
export function genData (el state) {
  let data = '{'
  // ...
  if (el.events) {
    data += `${genHandlers(el.events, false,state.warn)},`
  }
  if (el.nativeEvents) {
    data += `${genHandlers(el.nativeEvents, true, state.warn)},`
  }
  // ...
  return data
}

// 生成的data数据如下：
{
  // ...
  on: {"select": selectHandler}, // 自定义事件
  nativeOn: {"click": function($event) { // 原生事件
      return clickHandler($event)
    }
  }
  // ...
}
```

模板编译的最终目的是创建`render`函数供挂载的时候调用生成虚拟`DOM`，那么在挂载阶段， 如果被挂载的节点是一个组件节点，则通过 `createComponent` 函数创建一个组件 `vnode`。

```javascript
export function createComponent (
  Ctor: Class<Component> | Function | Object | void,
  data: ?VNodeData,
  context: Component,
  children: ?Array<VNode>,
  tag?: string
): VNode | Array<VNode> | void {
  // ...
  const listeners = data.on

  data.on = data.nativeOn

  // ...
  const name = Ctor.options.name || tag
  const vnode = new VNode(
    `vue-component-${Ctor.cid}${name ? `-${name}` : ''}`,
    data, undefined, undefined, undefined, context,
    { Ctor, propsData, listeners, tag, children },
    asyncFactory
  )

  return vnode
}
```

父组件给子组件的注册事件中，把自定义事件传给子组件，在子组件实例化的时候进行初始化；而浏览器原生事件是在父组件中处理。

##### initEvents函数分析

`initEvents`函数逻辑非常简单，首先在`vm`上新增`_events`属性并将其赋值为空对象，用来存储事件。接着，获取父组件注册的事件赋给`listeners`，如果`listeners`不为空，则调用`updateComponentListeners`函数，将父组件向子组件注册的事件注册到子组件的实例中。

```javascript
export function initEvents (vm: Component) {
  vm._events = Object.create(null)
  // init parent attached events
  const listeners = vm.$options._parentListeners
  if (listeners) {
    updateComponentListeners(vm, listeners)
  }
}
```

父组件既可以给子组件上绑定自定义事件，也可以绑定浏览器原生事件。这两种事件有着不同的处理时机，浏览器原生事件是由父组件处理，而自定义事件是在子组件初始化的时候由父组件传给子组件，再由子组件注册到实例的事件系统中。

也就是说：**初始化事件函数initEvents实际上初始化的是父组件在模板中使用v-on或@注册的监听子组件内触发的事件。**

最后分析了`initEvents`函数的具体实现过程，该函数内部首先在实例上新增了`_events`属性并将其赋值为空对象，用来存储事件。接着通过调用`updateComponentListeners`函数，将父组件向子组件注册的事件注册到子组件实例中的`_events`对象里。



#### 初始化阶段(initInjections) 【依赖注入】

这里所说的数据就是我们通常所写`data`、`props`、`watch`、`computed`及`method`，所以`inject`选项接收到注入的值有可能被以上这些数据所使用到，所以在初始化完`inject`后需要先初始化这些数据，然后才能再初始化`provide`，所以在调用`initInjections`函数对`inject`初始化完之后需要先调用`initState`函数对数据进行初始化，最后再调用`initProvide`函数对`provide`进行初始化。

```javascript
// initInjections函数的逻辑并不复杂，首先调用resolveInject把inject选项中的数据转化成键值对的形式赋给result
export function initInjections (vm: Component) {
  const result = resolveInject(vm.$options.inject, vm)
  // 在把result中的键值添加到当前实例上之前，会先调用toggleObserving(false)，而这个函数内部是把shouldObserve = false，这是为了告诉defineReactive函数仅仅是把键值添加到当前实例上而不需要将其转换成响应式。provide 和 inject 绑定并不是可响应的。这是刻意为之的。然而，如果你传入了一个可监听的对象，那么其对象的'属性'还是可响应的。
  if (result) {
    toggleObserving(false)
    Object.keys(result).forEach(key => {
      defineReactive(vm, key, result[key])
    }
    toggleObserving(true)
  }
}

export let shouldObserve: boolean = true
export function toggleObserving (value: boolean) {
  shouldObserve = value
}
```

##### resolveInject函数分析

`inject` 选项中的每一个数据`key`都是由其上游父级组件提供的，所以我们应该把每一个数据`key`从当前组件起，不断的向上游父级组件中查找该数据`key`对应的值，直到找到为止。如果在上游所有父级组件中没找到，那么就看在`inject` 选项是否为该数据`key`设置了默认值，如果设置了就使用默认值，如果没有设置，则抛出异常。

```javascript
export function resolveInject (inject: any, vm: Component): ?Object {
  if (inject) {
    const result = Object.create(null)
    const keys =  Object.keys(inject)

    // 然后获取当前inject 选项中的所有key，然后遍历每一个key，拿到每一个key的from属性记作provideKey，provideKey就是上游父级组件提供的源属性，然后	开启一个while循环，从当前组件起，不断的向上游父级组件的_provided属性中（父级组件使用provide选项注入数据时会将注入的数据存入自己的实例的_provided		属性中）查找，直到查找到源属性的对应的值，将其存入result中，
    for (let i = 0; i < keys.length; i++) {
      const key = keys[i]
      const provideKey = inject[key].from
      let source = vm
      while (source) {
        if (source._provided && hasOwn(source._provided, provideKey)) {
          result[key] = source._provided[provideKey]
          break
        }
        source = source.$parent
      }
      // 如果没有找到，那么就看inject 选项中当前的数据key是否设置了默认值，即是否有default属性，如果有的话，则拿到这个默认值
      if (!source) {
        if ('default' in inject[key]) {
          const provideDefault = inject[key].default
          result[key] = typeof provideDefault === 'function'
            ? provideDefault.call(vm)
            : provideDefault
        } else if (process.env.NODE_ENV !== 'production') {
          warn(`Injection "${key}" not found`, vm)
        }
      }
    }
    return result
  }
}
```

`inject`选项的规范化函数`normalizeInject`也进行了分析，`Vue`为用户提供了自由多种的写法，其内部是将各种写法最后进行统一规范化处理。

```javascript
function normalizeInject (options: Object, vm: ?Component) {
  const inject = options.inject
  if (!inject) return
  const normalized = options.inject = {}
  if (Array.isArray(inject)) {
    for (let i = 0; i < inject.length; i++) {
      normalized[inject[i]] = { from: inject[i] }
    }
  } else if (isPlainObject(inject)) {
    for (const key in inject) {
      const val = inject[key]
      normalized[key] = isPlainObject(val)
        ? extend({ from: key }, val)
        : { from: val }
    }
  } else if (process.env.NODE_ENV !== 'production') {
    warn(
      `Invalid value for option "inject": expected an Array or an Object, ` +
      `but got ${toRawType(inject)}.`,
      vm
    )
  }
}
```

#### 初始化阶段(initState) 【属性】

##### initState函数分析

```javascript
export function initState (vm: Component) {
  vm._watchers = []
  const opts = vm.$options
  if (opts.props) initProps(vm, opts.props)
  if (opts.methods) initMethods(vm, opts.methods)
  if (opts.data) {
    initData(vm)
  } else {
    observe(vm._data = {}, true /* asRootData */)
  }
  if (opts.computed) initComputed(vm, opts.computed)
  if (opts.watch && opts.watch !== nativeWatch) {
    initWatch(vm, opts.watch)
  }
}
```

##### 初始化props

在子组件内部，通过`props`选项来接收父组件传来的数据，在接收的时候可以这样写：

```js
// 写法一
props: ['name']

// 写法二
props: {
    name: String, // [String, Number]
}

// 写法三
props: {
    name:{
		type: String
    }
}
```

Vue给用户提供的props选项写法非常自由，根据Vue的惯例，写法虽多但是最终处理的时候肯定只处理一种写法。规范化属性数据代码如下：

```js
function normalizeProps (options, vm) {
  const props = options.props
  if (!props) return
  const res = {}
  let i, val, name
  if (Array.isArray(props)) {
    i = props.length
    while (i--) {
      val = props[i]
      if (typeof val === 'string') {
        name = camelize(val)
        res[name] = { type: null }
      } else if (process.env.NODE_ENV !== 'production') {
        warn('props must be strings when using array syntax.')
      }
    }
  } else if (isPlainObject(props)) {
    for (const key in props) {
      val = props[key]
      name = camelize(key)
      res[name] = isPlainObject(val)
        ? val
        : { type: val }
    }
  } else if (process.env.NODE_ENV !== 'production') {
    warn(
      `Invalid value for option "props": expected an Array or an Object, ` +
      `but got ${toRawType(props)}.`,
      vm
    )
  }
  options.props = res
}
```

###### initProps函数分析

```js
function initProps (vm: Component, propsOptions: Object) {
  // 父组件传入的真实props数据
  const propsData = vm.$options.propsData || {}
  // vm._props的指针，所有设置到props变量中的属性都会保存到vm._props中。
  const props = vm._props = {}
  // 指向vm.$options._propKeys的指针，缓存props对象中的key，将来更新props时只需遍历vm.$options._propKeys数组即可得到所有props的key。
  const keys = vm.$options._propKeys = []
  // 当前组件是否为根组件。
  const isRoot = !vm.$parent
  // root instance props should be converted
  if (!isRoot) {
    toggleObserving(false)
  }
  // 遍历props选项拿到每一对键值，先将键名添加到keys中，然后调用validateProp函数（关于该函数下面会介绍）校验父组件传入的props数据类型是否匹配并获取到传入的值value，然后将键和值通过defineReactive函数添加到props（即vm._props）中
  for (const key in propsOptions) {
    keys.push(key)
    const value = validateProp(key, propsOptions, propsData, vm)
    /* istanbul ignore else */
    if (process.env.NODE_ENV !== 'production') {
      const hyphenatedKey = hyphenate(key)
      if (isReservedAttribute(hyphenatedKey) ||
          config.isReservedAttr(hyphenatedKey)) {
        warn(
          `"${hyphenatedKey}" is a reserved attribute and cannot be used as component prop.`,
          vm
        )
      }
      defineReactive(props, key, value, () => {
        if (vm.$parent && !isUpdatingChildComponent) {
          warn(
            `Avoid mutating a prop directly since the value will be ` +
            `overwritten whenever the parent component re-renders. ` +
            `Instead, use a data or computed property based on the prop's ` +
            `value. Prop being mutated: "${key}"`,
            vm
          )
        }
      })
    } else {
      defineReactive(props, key, value)
    }
    // 添加完之后再判断这个key在当前实例vm中是否存在，如果不存在，则调用proxy函数在vm上设置一个以key为属性的代码，当使用vm[key]访问数据时，其实访问的是vm._props[key]
    if (!(key in vm)) {
      proxy(vm, `_props`, key)
    }
  }
  toggleObserving(true)
}
```

###### validateProp函数分析

```js
/*
* 校验传入属性是否符合要求
*
* key 遍历propOptions时拿到的每个属性名。
* propOptions 当前实例规范化后的props选项。
* propsData 父组件传入的真实props数据。
* vm 当前实例。
*/
export function validateProp (key,propOptions,propsData,vm) {
  const prop = propOptions[key]
  const absent = !hasOwn(propsData, key)
  let value = propsData[key]
  // boolean casting
  const booleanIndex = getTypeIndex(Boolean, prop.type)
  if (booleanIndex > -1) {
    if (absent && !hasOwn(prop, 'default')) {
      value = false
    } else if (value === '' || value === hyphenate(key)) {
      // only cast empty string / same name to boolean if
      // boolean has higher priority
      const stringIndex = getTypeIndex(String, prop.type)
      if (stringIndex < 0 || booleanIndex < stringIndex) {
        value = true
      }
    }
  }
  // check default value
  if (value === undefined) {
    value = getPropDefaultValue(vm, prop, key)
    // since the default value is a fresh copy,
    // make sure to observe it.
    const prevShouldObserve = shouldObserve
    toggleObserving(true)
    observe(value)
    toggleObserving(prevShouldObserve)
  }
  if (process.env.NODE_ENV !== 'production') {
    assertProp(prop, key, value, vm, absent)
  }
  return value
}
```

###### getPropDefaultValue函数分析

```js
/**
* 其作用是根据子组件props选项中的key获取其对应的默认值。
* vm 当前实例；
* prop 子组件props选项中的每个key对应的值；
* key 子组件props选项中的每个key；
*/
function getPropDefaultValue (vm, prop, key){
  // no default, return undefined
  if (!hasOwn(prop, 'default')) {
    return undefined
  }
  const def = prop.default
  // warn against non-factory defaults for Object & Array
  if (process.env.NODE_ENV !== 'production' && isObject(def)) {
    warn(
      'Invalid default value for prop "' + key + '": ' +
      'Props with type Object/Array must use a factory function ' +
      'to return the default value.',
      vm
    )
  }
  // the raw prop value was also undefined from previous render,
  // return previous default value to avoid unnecessary watcher trigger
  if (vm && vm.$options.propsData &&
    vm.$options.propsData[key] === undefined &&
    vm._props[key] !== undefined
  ) {
    return vm._props[key]
  }
  // call factory function for non-Function types
  // a value is Function if its prototype is function even across different execution context
  return typeof def === 'function' && getType(prop.type) !== 'Function'
    ? def.call(vm)
    : def
}
```

###### assertProp函数分析

```js
/*
* 其作用是校验父组件传来的真实值是否与prop的type类型相匹配，如果不匹配则在非生产环境下抛出警告。
* prop prop:prop选项;
* name:props中prop选项的key;
* value:父组件传入的propsData中key对应的真实数据；
* vm:当前实例；
* absent:当前key是否在propsData中存在，即父组件是否传入了该属性。
*/
function assertProp (prop,name,value,vm,absent) {
  if (prop.required && absent) {
    warn(
      'Missing required prop: "' + name + '"',
      vm
    )
    return
  }
  if (value == null && !prop.required) {
    return
  }
  let type = prop.type
  let valid = !type || type === true
  const expectedTypes = []
  if (type) {
    if (!Array.isArray(type)) {
      type = [type]
    }
    for (let i = 0; i < type.length && !valid; i++) {
      const assertedType = assertType(value, type[i])
      expectedTypes.push(assertedType.expectedType || '')
      valid = assertedType.valid
    }
  }
  if (!valid) {
    warn(
      `Invalid prop: type check failed for prop "${name}".` +
      ` Expected ${expectedTypes.map(capitalize).join(', ')}` +
      `, got ${toRawType(value)}.`,
      vm
    )
    return
  }
  const validator = prop.validator
  if (validator) {
    if (!validator(value)) {
      warn(
        'Invalid prop: custom validator check failed for prop "' + name + '".',
        vm
      )
    }
  }
}
```

##### 初始化methods

```js
function initMethods (vm, methods) {
  const props = vm.$options.props
  for (const key in methods) {
    if (process.env.NODE_ENV !== 'production') {
      // 遍历methods选项中的每一个对象，在非生产环境下判断如果methods中某个方法只有key而没有value，即只有方法名没有方法体时，抛出异常：提示用户方法未定义。
      if (methods[key] == null) {
        warn(
          `Method "${key}" has an undefined value in the component definition. ` +
          `Did you reference the function correctly?`,
          vm
        )
      }
      // 如果methods中某个方法名与props中某个属性名重复了，就抛出异常：提示用户方法名重复了。
      if (props && hasOwn(props, key)) {
        warn(
          `Method "${key}" has already been defined as a prop.`,
          vm
        )
      }
      // 如果methods中某个方法名如果在实例vm中已经存在并且方法名是以_或$开头的，就抛出异常：提示用户方法名命名不规范。
      if ((key in vm) && isReserved(key)) {
        warn(
          `Method "${key}" conflicts with an existing Vue instance method. ` +
          `Avoid defining component methods that start with _ or $.`
        )
      }
    }
    vm[key] = methods[key] == null ? noop : bind(methods[key], vm)
  }
}
```

##### 初始化data

```js
function initData (vm) {
    let data = vm.$options.data
    // 获取到用户传入的data选项，赋给变量data，同时将变量data作为指针指向vm._data，然后判断data是不是一个函数，如果是就调用getData函数获取其返回值，将其保存到vm._data中。如果不是，就将其本身保存到vm._data中
    data = vm._data = typeof data === 'function'
        ? getData(data, vm)
    : data || {}
    if (!isPlainObject(data)) {
        data = {}
        process.env.NODE_ENV !== 'production' && warn(
            'data functions should return an object:\n' +
            'https://vuejs.org/v2/guide/components.html##data-Must-Be-a-Function',
            vm
        )
    }
    // proxy data on instance
    const keys = Object.keys(data)
    const props = vm.$options.props
    const methods = vm.$options.methods
    let i = keys.length
    while (i--) {
        const key = keys[i]
        // data对象中的每一项，在非生产环境下判断data对象中是否存在某一项的key与methods中某个属性名重复，如果存在重复，就抛出警告：提示用户属性名重复。
        if (process.env.NODE_ENV !== 'production') {
            if (methods && hasOwn(methods, key)) {
                warn(
                    `Method "${key}" has already been defined as a data property.`,
                    vm
                )
            }
        }
        // 判断是否存在某一项的key与prop中某个属性名重复，如果存在重复，就抛出警告：提示用户属性名重复。
        if (props && hasOwn(props, key)) {
            process.env.NODE_ENV !== 'production' && warn(
                `The data property "${key}" is already declared as a prop. ` +
                `Use prop default value instead.`,
                vm
            )
        } else if (!isReserved(key)) {
            // 调用proxy函数将data对象中key不以_或$开头的属性代理到实例vm上，这样，我们就可以通过this.xxx来访问data选项中的xxx数据
            proxy(vm, `_data`, key)
        }
    }
    // observe data
    observe(data, true /* asRootData */)
}
```

##### 初始化computed

```js
function initComputed (vm: Component, computed: Object) {
    const watchers = vm._computedWatchers = Object.create(null)
    const isSSR = isServerRendering()

    for (const key in computed) {
        const userDef = computed[key]
        // 遍历computed选项中的每一项属性，首先获取到每一项的属性值，记作userDef，然后判断userDef是不是一个函数，如果是函数，则该函数默认为取值器getter，将其赋值给变量getter；如果不是函数，则说明是一个对象，则取对象中的get属性作为取值器赋给变量getter.
        const getter = typeof userDef === 'function' ? userDef : userDef.get
        if (process.env.NODE_ENV !== 'production' && getter == null) {
            warn(
                `Getter is missing for computed property "${key}".`,
                vm
            )
        }

        if (!isSSR) {
            // create internal watcher for the computed property.
            watchers[key] = new Watcher(
                vm,
                getter || noop,
                noop,
                computedWatcherOptions
            )
        }
		// 判断当前循环到的的属性名是否存在于当前实例vm上，如果存在，则在非生产环境下抛出警告；如果不存在，则调用defineComputed函数为实例vm上设置计算属性。
        if (!(key in vm)) {
            defineComputed(vm, key, userDef)
        } else if (process.env.NODE_ENV !== 'production') {
            if (key in vm.$data) {
                warn(`The computed property "${key}" is already defined in data.`, vm)
            } else if (vm.$options.props && key in vm.$options.props) {
                warn(`The computed property "${key}" is already defined as a prop.`, vm)
            }
        }
    }
}
```

###### defineComputed函数分析

```js
const sharedPropertyDefinition = {
  enumerable: true,
  configurable: true,
  get: noop,
  set: noop
}
/*
* 作用是为target上定义一个属性key，并且属性key的getter和setter根据userDef的值来设置
* target
* key
* userDef
*/
export function defineComputed (target,key,userDef) {
  const shouldCache = !isServerRendering()
  if (typeof userDef === 'function') {
    sharedPropertyDefinition.get = shouldCache
      ? createComputedGetter(key)
      : userDef
    sharedPropertyDefinition.set = noop
  } else {
    sharedPropertyDefinition.get = userDef.get
      ? shouldCache && userDef.cache !== false
        ? createComputedGetter(key)
        : userDef.get
      : noop
    sharedPropertyDefinition.set = userDef.set
      ? userDef.set
      : noop
  }
  if (process.env.NODE_ENV !== 'production' &&
      sharedPropertyDefinition.set === noop) {
    sharedPropertyDefinition.set = function () {
      warn(
        `Computed property "${key}" was assigned to but it has no setter.`,
        this
      )
    }
  }
  Object.defineProperty(target, key, sharedPropertyDefinition)
}
```

###### createComputedGetter函数分析

该函数是一个高阶函数，其内部返回了一个`computedGetter`函数，所以其实是将`computedGetter`函数赋给了`sharedPropertyDefinition.get`。当获取计算属性的值时会执行属性的`getter`，而属性的`getter`就是 `sharedPropertyDefinition.get`，也就是说最终执行的 `computedGetter`函数。

```js
function createComputedGetter (key) {
    return function computedGetter () {
        const watcher = this._computedWatchers && this._computedWatchers[key]
        if (watcher) {
            // 收集依赖
            watcher.depend()
            // 将计算结果返回
            return watcher.evaluate()
        }
    }
}
```

######  depend和evaluate

```js
const computedWatcherOptions = { computed: true }
watchers[key] = new Watcher(
    vm,
    getter || noop,
    noop,
    computedWatcherOptions
)

export default class Watcher {
    constructor (vm,expOrFn,cb,options,isRenderWatcher) {
        if (options) {
            // ...
            this.computed = !!options.computed
            // ...
        } else {
            // ...
        }

        this.dirty = this.computed // for computed watchers
        if (typeof expOrFn === 'function') {
            this.getter = expOrFn
        }

        if (this.computed) {
            this.value = undefined
            this.dep = new Dep()
        }
    }

    evaluate () {
        if (this.dirty) {
            this.value = this.get()
            this.dirty = false
        }
        return this.value
    }

    /**
     * Depend on this watcher. Only for computed property watchers.
     */
    depend () {
        if (this.dep && Dep.target) {
            this.dep.depend()
        }
    }

    update () {
        if (this.computed) {
            if (this.dep.subs.length === 0) {
                this.dirty = true
            } else {
                this.getAndInvoke(() => {
                    this.dep.notify()
                })
            }
        }
    }

    getAndInvoke (cb: Function) {
        const value = this.get()
        if (
            value !== this.value ||
            // Deep watchers and watchers on Object/Arrays should fire even
            // when the value is the same, because the value may
            // have mutated.
            isObject(value) ||
            this.deep
        ) {
            // set new value
            const oldValue = this.value
            this.value = value
            this.dirty = false
            if (this.user) {
                try {
                    cb.call(this.vm, value, oldValue)
                } catch (e) {
                    handleError(e, this.vm, `callback for watcher "${this.expression}"`)
                }
            } else {
                cb.call(this.vm, value, oldValue)
            }
        }
    }
}
```

可以看到，在实例化`Watcher`类的时候，第四个参数传入了一个对象`computedWatcherOptions = { computed: true }`，该对象中的`computed`属性标志着这个`watcher`实例是计算属性的`watcher`实例，即`Watcher`类中的`this.computed`属性，同时类中还定义了`this.dirty`属性用于标志计算属性的返回值是否有变化，计算属性的缓存就是通过这个属性来判断的，每当计算属性依赖的数据发生变化时，会将`this.dirty`属性设置为`true`，这样下一次读取计算属性时，会重新计算结果返回，否则直接返回之前的计算结果。

当调用`watcher.depend()`方法时，会将读取计算属性的那个`watcher`添加到计算属性的`watcher`实例的依赖列表中，当计算属性中用到的数据发生变化时，计算属性的`watcher`实例就会执行`watcher.update()`方法，在`update`方法中会判断当前的`watcher`是不是计算属性的`watcher`，如果是则调用`getAndInvoke`去对比计算属性的返回值是否发生了变化，如果真的发生变化，则执行回调，通知那些读取计算属性的`watcher`重新执行渲染逻辑。

当调用`watcher.evaluate()`方法时，会先判断`this.dirty`是否为`true`，如果为`true`，则表明计算属性所依赖的数据发生了变化，则调用`this.get()`重新获取计算结果最后返回；如果为`false`，则直接返回之前的计算结果。

![img](https://vue-js.com/learn-vue/assets/img/2.3828fb66.png)

##### 初始化watch

在函数内部会遍历`watch`选项，拿到每一项的`key`和对应的值`handler`。然后判断`handler`是否为数组，如果是数组则循环该数组并将数组中的每一项依次调用`createWatcher`函数来创建`watcher`；如果不是数组，则直接调用`createWatcher`函数来创建`watcher`。

```js
function initWatch (vm, watch) {
  for (const key in watch) {
    const handler = watch[key]
    if (Array.isArray(handler)) {
      for (let i = 0; i < handler.length; i++) {
        createWatcher(vm, key, handler[i])
      }
    } else {
      createWatcher(vm, key, handler)
    }
  }
}
```

###### createWatcher

```js
/*
* vm:当前实例；
* expOrFn:被侦听的属性表达式
* handler:watch选项中每一项的值
* options:用于传递给vm.$watch的选项对象
*/
function createWatcher (
  vm: Component,
  expOrFn: string | Function,
  handler: any,
  options?: Object
) {
  if (isPlainObject(handler)) {
    options = handler
    handler = handler.handler
  }
  if (typeof handler === 'string') {
    handler = vm[handler]
  }
  return vm.$watch(expOrFn, handler, options)
}
```

生命周期初始化阶段所调用的第五个初始化函数——`initState`。该初始化函数内部总共初始化了5个选项，分别是：`props`、`methods`、`data`、`computed`和`watch`。

这5个选项的初始化顺序不是任意的，而是经过精心安排的。只有按照这种顺序初始化我们才能在开发中在`data`中可以使用`props`，在`watch`中可以观察`data`和`props`。

这5个选项中的所有属性最终都会被绑定到实例上，这也就是我们为什么可以使用`this.xxx`来访问任意属性。同时正是因为这一点，这5个选项中的所有属性名都不应该有所重复，这样会造成属性之间相互覆盖。

#### 模板编译阶段

#### 挂载阶段

#### 销毁阶段