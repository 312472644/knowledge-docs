### **变化侦测篇**

#### Object的变化侦测

##### 1.使Object数据变得“可观测”

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

##### 2.依赖收集

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

##### 3.依赖到底是谁

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

##### 4、不足之处

虽然我们通过`Object.defineProperty`方法实现了对`object`数据的可观测，但是这个方法仅仅只能观测到`object`数据的取值及设置值，当我们向`object`数据里添加一对新的`key/value`或删除一对已有的`key/value`时，它是无法观测到的，导致当我们对`object`数据添加或删除值时，无法通知依赖，无法驱动视图进行响应式更新。

##### 5、总结

其整个流程大致如下：

1. `Data`通过`observer`转换成了`getter/setter`的形式来追踪变化。
2. 当外界通过`Watcher`读取数据时，会触发`getter`从而将`Watcher`添加到依赖中。
3. 当数据发生了变化时，会触发`setter`，从而向`Dep`中的依赖（即Watcher）发送通知。
4. `Watcher`接收到通知后，会向外界发送通知，变化通知到外界后可能会触发视图更新，也有可能触发用户的某个回调函数等。

#### Array的变化侦测

##### 1.使Array型数据可观测

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

##### 2.收集依赖

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

##### 3.如何通知依赖

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

##### 4.深度侦测

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

##### 5.不足之处

对于数组变化侦测是通过拦截器实现的，也就是说只要是通过数组原型上的方法对数组进行操作就都可以侦测到，但是别忘了，我们在日常开发中，还可以通过数组的下标来操作数据，如下：

```javascript
let arr = [1,2,3]
arr[0] = 5;       // 通过数组下标修改数组中的数据
arr.length = 0    // 通过修改数组长度清空数组
```

而使用上述例子中的操作方式来修改数组是无法侦测到的。 同样，`Vue`也注意到了这个问题， 为了解决这一问题，`Vue`增加了两个全局API:`Vue.set`和`Vue.delete`，这两个API的实现原理将会在后面学习全局API的时候说到。