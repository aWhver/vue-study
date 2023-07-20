# Vue实例化过程

> 源码位置 \`core/instance/init.js\`

#### 1、选项合并

Vue构造函数的options包含了很多选项，开发者可根据自己的需求传入自己的需要的属性，Vue初始化的时候会根据传入的选项和默认的选项做一次合并，保证Vue内部实例化所需要的属性都存在

#### 2、初始化生命周期

主要对`$parent`、`$root`、`$children`、`$refs`等做初始化

#### 3、对事件做初始化

对自定义事件做初始化

#### 4、初始化渲染

设置DOM更新函数（createElement），处理插槽相关信息，将`$attrs`和`$listeners`处理为响应式数据

#### 5、调用钩子beforeCreate

#### 6、初始化Inject

对祖辈传下来的inject数据做响应式处理

#### 7、初始内部状态

对data、props、methods、computed、watch等初始化

#### 8、初始化Provide

处理向下传递的provide数据

#### 9、调用钩子created

#### 10、执行挂载函数$moun
