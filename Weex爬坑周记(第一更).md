# Weex爬坑周记 1

## 学习目的
 1. 达到开发一次三端运行的目的
 2. 掌握前端开发技能
 
## Weex环境搭建
略

##VSCode

 1. 插件：vscode-weex,weex,weex-debugger,weex-doctor,weex-lang,weex-new-project,weex-run,Vetur
 2. 格式化时无法识别文件，右下角选择对应文件类型

![](/pics/vscode_file.png)
 
## Js

 1. Objct.assign()复制多个对象到一个对象
 2. ``es6新语法，可用于：普通字符串、多行字符串、嵌入变量
 3. 函数参数，简单数据类型深拷贝，对象数据类型浅拷贝。要改变data内属性，可把this.$data作为参数
 4. 作用域链：访问变量顺序
 
## 页面搭建

页面组成：
1、weex基本控件标签
2、weex通用样式+flexbox
3、数据驱动vue

### vue

#### 语法

 1. data只能是简单对象
 2. 绑定文本,Mastach语法"{{}}"
 3. 绑定属性v-bind，简写:XXX(XXX表示属性)
 4. 绑定点击事件v-on:click="XXX"，简写@click="XXX"
 5. v-for指令可以添加索引参数，如:v-for="(item,i) in items"，i表示索引
 6. computed属性，vue实例data内属性运算
 7. watch属性，监听data内属性变化
 8. 其他指令：v-for,v-if,v-show,v-model
 
>vue实例内使用data内属性，需要有this关键字，如this.XXX。否则，会XXX is not defined

#### vue-router
`<router-view>`页面占位标签
router.push()方法切换页面，go()另有用处

> 在router.js中调用Vue.use(Router)在web端会Vue is not defined，app端正常。猜测此时Vue还未挂载到window下，改为在entry.js调用Vue.use(Router)

## Weex
 1. 在移动端flex:{number}要和width或height一起使用才有效
 2. storage,picker组件回调返回的结果是data，记：返回数据(data)
 3. weex run web，可以用playground扫码，weex debug src/index.vue，扫码菊花一直转

## webpack

 1. config/config.js配置useEslint:false，可禁用Eslint
