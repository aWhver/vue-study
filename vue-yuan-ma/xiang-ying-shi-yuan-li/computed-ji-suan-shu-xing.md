# computed计算属性

> 计算属性通常用来处理一些复杂的逻辑处理。虽然模版内可以写表达式，但初衷是用来做简单计算的，如果模版内的表达式太过复杂会让代码变得难以维护，所以如果表达式逻辑复杂的话尽量用计算属性，其次计算属性有缓存作用

### 初始化

初始化执行地方和[<mark style="color:orange;">`watch`</mark>](https://juntong.gitbook.io/vuejs/vue-yuan-ma/shu-ju-qu-dong/watch-jian-ting#chu-shi-hua)监听器一样，执行时机在watch之前。在vue实例化的过程中，会执行原型上`_init`方法，该方法会依次执行一系列初始化过程，其中一项就是`initState`，该方法就对我们常用的data，props，computed等做了初始化。initState的代码路径在`src/core/instance/state.js`，在这内部就会判断是否传入了computed对象，然后对其初始化

{% code lineNumbers="true" %}
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
  // 初始化计算属性
  if (opts.computed) initComputed(vm, opts.computed)
  // 初始化watch监听属性
  if (opts.watch && opts.watch !== nativeWatch) {
    initWatch(vm, opts.watch)
  }
}
```
{% endcode %}

### 创建监听

计算属性和watch监听器一样，同样会使用Watcher构造函数新建一个watcher对象，同时在vue实例下新建一个`_computedWatchers`对象，用来存储每一个计算属性的watcher,该对象的作用是对计算属性取值时用来获取相对应的watcher来进行表达式的计算和结果获取

{% code lineNumbers="true" %}
```javascript
// src/core/instance/state.js
function initComputed (vm: Component, computed: Object) {
  // 新建一个watcher存储对象，后面用来取值和更新
  const watchers = vm._computedWatchers = Object.create(null)
  // computed properties are just getters during SSR
  const isSSR = isServerRendering()
  
  // 遍历computed对象，为每一个计算属性新建一个watcher对象
  for (const key in computed) {
    const userDef = computed[key]
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

    // component-defined computed properties are already defined on the
    // component prototype. We only need to define computed properties defined
    // at instantiation here.
    // 判断属性是否在vue实例上,避免和data属性冲突
    if (!(key in vm)) {
      // 对计算属性进行拦截处理
      defineComputed(vm, key, userDef)
    }
  }
 } 
```
{% endcode %}

### 属性劫持处理

* 首先会判断是否为服务端渲染，是则每次都会重新计算结果，不会对结果做缓存，反之则会做缓存处理，后续会根据是否为服务端这个标志为`set`函数设置不同的劫持函数
* 判断属性值类型是否为函数，函数则将createComputedGetter的返回值作为`get`函数
* 判断属性值类型是否为对象，对象则将`set`函数设置为对象的set函数，`get`函数则根据是否是服务端渲染和对象的`cache`值是否为`true`,来决定用`createComputedGetter`的返回值做为`get`函数还是对象的get函数
* 使用`Object.defineProperty`对vue实例进行劫持处理

{% code lineNumbers="true" %}
```javascript
export function defineComputed (
  target: any,
  key: string,
  userDef: Object | Function
) {
  // 判断是否是服务端渲染 
  const shouldCache = !isServerRendering()
  // 判断属性值为函数
  if (typeof userDef === 'function') {
    sharedPropertyDefinition.get = shouldCache
      ? createComputedGetter(key)
      : createGetterInvoker(userDef)
    sharedPropertyDefinition.set = noop
  } else {
    // 判断属性值为对象
    sharedPropertyDefinition.get = userDef.get
      ? shouldCache && userDef.cache !== false
        ? createComputedGetter(key)
        : createGetterInvoker(userDef.get)
      : noop
    sharedPropertyDefinition.set = userDef.set || noop
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
  // 对vue上的计算属性进行劫持处理
  Object.defineProperty(target, key, sharedPropertyDefinition)
}
```
{% endcode %}

### 劫持函数

通过属性`key`获取对应的`watcher`对象，判断watcher上的`dirty`是否为`true`，为`true`则执行对象下的`evaluate`方法更新`watcher.value`，`dirty`属性用来判断监听的值是否发生了变化，然后重新计算属性值；判断`Dep.target`当前是否存在Watcher对象，有的话，进行依赖收集，并将`dirty`置为`false`，依赖发生变化时，`watcher`的`update`方法调用将`dirty`置为`true`下次获取计算属性时需要重新计算watcher.value；返回watcher.value作为计算结果

{% code lineNumbers="true" %}
```javascript
function createComputedGetter (key) {
  return function computedGetter () {
    // 获取上面创建的watcher对象 
    const watcher = this._computedWatchers && this._computedWatchers[key]
    if (watcher) {
      // dirty表示是否需要重新计算结果，执行完将dirty置为false
      if (watcher.dirty) {
        // 如果表达式里有响应式数据则会触发get函数进行依赖收集
        // 将Dep对象添加到当前依赖的Watcher对象中 
        watcher.evaluate()
      }
      // 依赖收集，后续值发生变化时，update函数将dirty置为true
      // note: 这个Dep.target是当前渲染的watcher不是上面的计算属性watcher
      if (Dep.target) {
        // 将watcher对象添加到依赖的Dep订阅列表中 
        watcher.depend()
      }
      return watcher.value
    }
  }
}
```
{% endcode %}

### 总结

vue初始化时会对传入的`data`做响应式处理，同时对每一个属性建立一个`Dep`对象，用来存储`watcher`列表；然后初始化computedWatcher，访问计算属性时会根据dirty是否执行表达式计算逻辑，当计算属性内访问了响应式数据，会调用`Dep.depend`会触发依赖收集，将Dep对象添加到watcher对象中；如果当前渲染`Watcher`存在，会调用computedWatcher.depend，将当前watcher对象添加到Dep订阅列表中。当访问的响应式数据更新时，Dep对象下的watcher对象执行`update`方法，更新`dirty`，访问计算属性时，computedWatcher执行`evaluate`方法重新计算表达式
