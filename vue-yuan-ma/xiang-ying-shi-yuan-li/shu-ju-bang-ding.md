---
description: 分析vue如何实现数据双向绑定的
---

# 数据绑定

* vue的双向绑定原理是通过`Object.defineProperty`这个API来实现的
* 建立了一套以`发布订阅`为主的依赖收集更新机制，主要是`Dep`类和`Watcher`类(后面分析)

### Object.defineProperty

`Object.defineProperty`的作用是对一个对象新增的属性或者原有的属性做劫持，并返回该对象，当访问或者设置被劫持的属性时会触发劫持函数`set`、`get`

```javascript
const proxyObj = Object.defineProperty(target, prop, descriptor);
```

`target`是要在其上定义属性的对象；`prop`则是要被定义或者修改的属性；`descriptor`是属性的描述符

描述符`descriptor`是关键的部分，类型为一个对象，里面定义了可选键值，具体可参考[<mark style="color:orange;">文档</mark>](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global\_Objects/Object/defineProperty)<mark style="color:orange;">。</mark>这里我们关注的是`get`和`set`函数，`get`函数访问属性的时候触发，`set`函数设置属性值的时候触发。

当属性设置了这2个可选键值时，我们就可以通过劫持者2个函数对属性值做额外处理，而vue就是通过`get`函数进行依赖收集，建立数据的依赖关系，`set`进行依赖分发，更新通知



### 初始化时机initData

当vue构造函数实例化时会执行`_init`方法，这个过程中会做一系列事情，包括生命周期、事件、渲染、组件内部状态等等初始化的操作。其中的内部状态初始化，也就是`initState`，在这个函数里会对我们传入的`data`数据做响应式处理。initState的文件路径在`src/core/instance/state.js`

```javascript
export function initState (vm: Component) {
  vm._watchers = []
  const opts = vm.$options
  // 对props做响应式处理
  if (opts.props) initProps(vm, opts.props)
  // 对方法初始化，绑定到vm实例上
  if (opts.methods) initMethods(vm, opts.methods)
  // 对data做响应式处理
  if (opts.data) {
    initData(vm)
  } else {
    observe(vm._data = {}, true /* asRootData */)
  }
  // 初始化计算属性
  if (opts.computed) initComputed(vm, opts.computed)
  // 初始化watch监听属性
  if (opts.watch && opts.watch !== nativeWatch) {
    initWatch(vm, opts.watch)
  }
}
```

这里重点分析`initData`，[<mark style="color:green;">**initComputed**</mark>](https://juntong.gitbook.io/vuejs/vue-yuan-ma/shu-ju-qu-dong/computed-ji-suan-shu-xing)和[<mark style="color:green;">**initWatch**</mark>](https://juntong.gitbook.io/vuejs/vue-yuan-ma/shu-ju-qu-dong/watch-jian-ting)后面单独分析。

```javascript
function initData (vm: Component) {
  let data = vm.$options.data
  // data赋值给vm._data，后续用作属性代理用，后面proxy具体分析
  data = vm._data = typeof data === 'function'
    ? getData(data, vm)
    : data || {}
  if (!isPlainObject(data)) {
    data = {}
    process.env.NODE_ENV !== 'production' && warn(
      'data functions should return an object:\n' +
      'https://vuejs.org/v2/guide/components.html#data-Must-Be-a-Function',
      vm
    )
  }
  // proxy data on instance
  const keys = Object.keys(data)
  const props = vm.$options.props
  const methods = vm.$options.methods
  let i = keys.length
  // 遍历data属性，对每个属性做一层代理
  while (i--) {
    const key = keys[i]
    if (process.env.NODE_ENV !== 'production') {
      if (methods && hasOwn(methods, key)) {
        warn(
          `Method "${key}" has already been defined as a data property.`,
          vm
        )
      }
    }
    if (props && hasOwn(props, key)) {
      process.env.NODE_ENV !== 'production' && warn(
        `The data property "${key}" is already declared as a prop. ` +
        `Use prop default value instead.`,
        vm
      )
    } else if (!isReserved(key)) { // 检查属性key是否以$/_开头，在vue里$和_开头的属于关键字
      // 属性代理
      proxy(vm, `_data`, key)
    }
  }
  // observe data
  // 对data进行响应式处理
  observe(data, true /* asRootData */)
}
```

这里主要做了2件事：

* data赋值给vm._data，_遍历data将属性进行代理，用`proxy`将`vm._data.xx`代理到`vm.xx`，这也是我们为什么可以在组件直接通过`this.xx`访问和设置data的原因
* 调用`observer`对整个`data`进行响应式处理并观测其变化



### proxy

```javascript
const sharedPropertyDefinition = {
  enumerable: true,
  configurable: true,
  get: noop,
  set: noop
}

export function proxy (target: Object, sourceKey: string, key: string) {
  // 访问目标对象属性实际就是访问目标对象下的sourceKey对象的属性
  // 这里的sourceKey指上面设置到vm上的_data对象
  sharedPropertyDefinition.get = function proxyGetter () {
    return this[sourceKey][key]
  }
  sharedPropertyDefinition.set = function proxySetter (val) {
    this[sourceKey][key] = val
  }
  //对目标对象进行代理
  Object.defineProperty(target, key, sharedPropertyDefinition)
}
```

`proxy`的作用就是将`target[sourceKey][key]`的读写变成了`target[key]`的读写。这里结合上面调用`proxy`的地方一起看就是将vm实例上`_data`对象属性都代理到vm上，当访问vm.xx的时候实际就是访问vm.\_data.xx，也就能够访问到我们定义的`data`数据了



### observe和Observer

&#x20;observe

observer的作用就是创建一个观察者实例，用来检测数据变化，他是一个单例模式，如果实例不存在且满足特定条件就创建，存在就直接返回

```javascript
export function observe (value: any, asRootData: ?boolean): Observer | void {
  if (!isObject(value) || value instanceof VNode) {
    return
  }
  let ob: Observer | void
  if (hasOwn(value, '__ob__') && value.__ob__ instanceof Observer) {
    // 这是判断数据是否是响应式的依据，只有响应式数据才会添加__ob__属性 
    ob = value.__ob__
  } else if (
    shouldObserve &&
    !isServerRendering() &&
    (Array.isArray(value) || isPlainObject(value)) &&
    Object.isExtensible(value) &&
    !value._isVue
  ) {
    // 创建观察者实例
    ob = new Observer(value)
  }
  if (asRootData && ob) {
    ob.vmCount++
  }
  return ob
}
```

#### Observer

这是一个观察者类，作用是对数据建立依赖和添加set和get劫持函数，收集依赖关系用于派发更新。

* 首先建立一个`Dep`对象，用来收集与之相关的`Watcher`订阅列表，后续用于派发更新
* 将观察者实例绑定到数据\_\_ob\_\_的属性上，作为识别数据是否是响应式数据的标识，有2个作用：
  1. 数据追踪。当数据发生变化时，`__ob__`会通知相关的`Watcher`进行更新
  2. 防止重复转换。Vue会检测一个对象是否有`__ob__`属性，有的话则说明已经是响应式数据了，就不会进行重复转换，其实就是`observe`的功能
* 对数据进行类型判断，对象就遍历属性调用`defineReactive`对属性做`get/set`劫持处理，数组则会重写数组API的7个方法，然后对数组进行遍历调用`observe`对数组每一项做响应式处理

```javascript
export class Observer {
  value: any;
  dep: Dep;
  vmCount: number; // number of vms that have this object as root $data

  constructor (value: any) {
    this.value = value
    // 新建Dep对象，用来收集与之相关的watcher订阅列表
    this.dep = new Dep()
    this.vmCount = 0
    // 将观察者实例绑定到value的__ob__属性上，这个属性可以用来判断数据是否为响应式数据
    def(value, '__ob__', this)
    // 进行数据类型判断，数组的话重写7个数组方法用来做派发更新，否则数组API对数据的修改无法实现响应式
    if (Array.isArray(value)) {
      if (hasProto) {
        protoAugment(value, arrayMethods)
      } else {
        copyAugment(value, arrayMethods, arrayKeys)
      }
      this.observeArray(value)
    } else {
      // 对对象做响应式处理 
      this.walk(value)
    }
  }

  walk (obj: Object) {
    const keys = Object.keys(obj)
    for (let i = 0; i < keys.length; i++) {
      // 设置每个数据的get/set劫持函数 
      defineReactive(obj, keys[i])
    }
  }

  /**
   * Observe a list of Array items.
   */
  observeArray (items: Array<any>) {
    for (let i = 0, l = items.length; i < l; i++) {
      // 对数据每一项做响应式处理 
      observe(items[i])
    }
  }
}
```

### defineReactive

这个函数就是Vue很核心的东西了，整个Vue的运转都要依赖它才能完成。该方法在`Observer`被调用，它的作用就是对每个数据进行get/set劫持，将其变成响应式数据。

* 新建依赖收集对象`Dep`，用作存储Watcher订阅列表
* 递归处理子对象，确保复杂的数据结构也都是响应式数据
* `get`函数中判断当前是否有正在渲染的`Watcher`实例，有的话则进行依赖收集，将当前`Watcher`添加到订阅列表中
* 数据变更时，触发`set`函数，`Dep`对象通知订阅列表中的`Watcher`进行更新

```javascript
export function defineReactive (
  obj: Object,
  key: string,
  val: any,
  customSetter?: ?Function,
  shallow?: boolean
) {
  // 新建依赖收集对象，用作储存Watcher订阅列表
  const dep = new Dep()

  const property = Object.getOwnPropertyDescriptor(obj, key)
  if (property && property.configurable === false) {
    return
  }

  const getter = property && property.get
  const setter = property && property.set
  if ((!getter || setter) && arguments.length === 2) {
    val = obj[key]
  }

  // 递归处理对象属性，保证所有的数据都是响应式数据
  let childOb = !shallow && observe(val)
  Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    get: function reactiveGetter () {
      const value = getter ? getter.call(obj) : val
      if (Dep.target) {
        // 进行依赖收集(将Watcher添加到Dep的订阅列表中)
        dep.depend()
        // 处理对象深层次的属性依赖收集
        if (childOb) {
          childOb.dep.depend()
          // 处理数组的依赖收集
          if (Array.isArray(value)) {
            dependArray(value)
          }
        }
      }
      return value
    },
    set: function reactiveSetter (newVal) {
      const value = getter ? getter.call(obj) : val
      if (newVal === value || (newVal !== newVal && value !== value)) {
        return
      }
      if (process.env.NODE_ENV !== 'production' && customSetter) {
        customSetter()
      }
      if (getter && !setter) return
      if (setter) {
        setter.call(obj, newVal)
      } else {
        val = newVal
      }
      // 对新值做响应式处理，建立新的依赖关系
      childOb = !shallow && observe(newVal)
      // 数据变更，通知Watcher更新
      dep.notify()
    }
  })
}
```

### 总结

Vue的数据响应式绑定就是利用了`Object.defineProperty`这个API对数据的`get`和`set`进行劫持，然后在`get`函数里进行依赖收集，`set`函数里进行派发更新，最后我们再读写数据的时候可以做额外逻辑
