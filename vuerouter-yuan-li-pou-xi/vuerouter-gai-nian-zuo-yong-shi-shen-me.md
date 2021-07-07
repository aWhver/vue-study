# Vue-router概念？作用是什么？设计思路是什么？

### Vue-router的定义？
&emsp;&emsp;Vue-router是一个Vue插件，官方的路由管理器。作为一个应用/网页，内容肯定不止一个页面，开发一个单页应用就需要用户的操作跳转到相应的展示页面，因此需要一个东西来管理这些页面，Vue-router就是这么一个东西。

### Vue-router的作用？
&emsp;&emsp;主要就是用作页面切换。根据浏览器地址的变化，在不刷新浏览器的情况下，实现内容的过渡，显示相应的页面内容。

### 设计思路

- 首先Vue-router是一个插件，按照Vue官方对插件的定义就是需要一个具有 `install` 方法的 `对象` 或者 `class类`、`构造函数` ，install方法接受一个 `Vue构造函数` 的参数。
- 我们使用的时候是以 `new VueRouter()` 的形式，注册路由表，因此 `Vue-router` 是一个 `class类` 或者 `构造函数`。
- 通过 `Vue.use` 方法注入路由后，我们就直接可以使用 `router-link`、 `router-view` 这2个组件，可以知道vue-router在注册的时候的在install方法里注册了这2个全局组件
- 在`class类` 或者 `构造函数`里实现路由监听，即 `hashchange` 监听，根据当前路由和已注册的路由表进行匹配
- 因为 `router-view` 是路由页面的显示的逻辑，因此在该组件里实现路由匹配
- 路由嵌套。需要记录当前router-view的嵌套深度，同时建立一个matched数组用来表示每个深度需要显示的路由信息，最后根据深度从数组里取出应该显示的路由信息，得到里面的 `component` 选项用作渲染
    - 深度可以在 `router-view` 的 `render` 函数内通过this获取 `vnode.data` 给自己设置 `routerView=true`，之后根据 while 迭代自身是否有 `$parent`，再判断变量下的 `vnode.data.routerView` 是否为true，是的话深度 **+1**，再设置更新变量 `parent` 为新的parent.$parent
    - vue-router类内通过遍历路由表和监听路由变化实现mathced数组的填充（需要将matched数组变成响应式数据，使用了响应式数据的组件才可以刻组件形成依赖关系，实现数据变化，视图重新渲染）
