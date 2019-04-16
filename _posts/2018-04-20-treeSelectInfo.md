---
layout:     post
title:      react 组件之间的通信
subtitle:   父组件调用子组件的方法
date:       2018年4月21日23:18:14
author:     darkceey
header-img: img/about_desk.jpg
catalog: true
tags:
    - 技术
---

> 最近项目前端使用react框架，使用公司牛人研发的NornJ(https://github.com/joe-sky/nornj-cli/tree/master/templates/react-mobx)作为加载引擎，在使用的过程遇到了两个问题，双向绑定和父组件与子组件之间的通信。下面一一叙述；

### 双向绑定

在store里定义一个observable类型的变量a，a只要为observable类型，当a的属性值发生变化，则会重新加render，即会重新绑定；

### toJS的作用

mobx里有个方法名为toJS，可以将observable类型转换为传统的js类型，比如使用toJS可以让observableArray转换为传统的Array类型，特别好用；

### 组件之间的通信

今天遇到了一个问题，在一个父组件中自定义了treeSelect组件，父组件是一个条件选择区域，如下图所示：
![mark](http://p6i3n2fdv.bkt.clouddn.com/blog/180421/HlD3hC0cB0.png?imageslim)
这个下拉选择框是ant.desgin的treeSelect组件，在父组件中自定义的引进来的，看代码
HTML：
![mark](http://p6i3n2fdv.bkt.clouddn.com/blog/180421/jDgF5im08B.png?imageslim)

container：
![mark](http://p6i3n2fdv.bkt.clouddn.com/blog/180421/iFFjD0gl63.png?imageslim)


这里，我需要点击重置，将treeSelect置空，重置按钮在父组件中，这时就需要父组件调用子组件中的方法；

具体实现：
在子组件上添加ref定义，即
![mark](http://p6i3n2fdv.bkt.clouddn.com/blog/180421/11CLDfIChC.png?imageslim)

并在子组件中定义父组件清空需要的方法，即resetSubTreeSelect方法；

父组件中的重置按钮点击时，就调用子组件中的重置方法，具体的实现是在父组件要调用的方法中添加
this.refs.treeSelect.wrappedInstance,后面跟子组件中定义的方面，点击重置时，就会进入到子组件中，进行置空；
![mark](http://p6i3n2fdv.bkt.clouddn.com/blog/180421/A69fecBkIK.png?imageslim)

总结：
看起来比较简单，但react真正的精髓组件之间的通信和双向绑定这二者必是重点，因为花了好长时间才搞定，所以特此记录下~