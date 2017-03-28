#属性篇  
* 属性篇，分为属性的定义及属性的获取；
* 重点掌握<declare-styleable>标签下<attr name="" format="">中format标签的定义。format的形式包括<b>refrence、string、float、dimension、fraction、color、boolean、integer、enum、flag</b>。属性定义时，可指定多种类型如“refrence|color”、“float|fraction”等；
* 主要涉及的类包括attrs.xml、TypedArray、TypedValue等
* 使用时可参考sdk/android/view/View.class、sdk/res/values/attrs.xml等文件的写法  
# 工具篇
* 主要是涉及到系统常量的定义，以及触摸时间的处理。如View拖拽、计算滑动速度、手势处理等；
* 主要涉及<b>Configuration.class、ViewConfiguration.class、GestureDetector.class、VelocityTracker.class、Scroller.class、ViewDragHelper.class</b> 这6个工具类；
* [自定义View系列教程01--常用工具介绍](http://www.2cto.com/kf/201605/506149.html)  
 [ViewDragHelper源码浅析 ](http://blog.csdn.net/CsL664867596/article/details/45332039)  
#方法篇  
* 主要涉及<b>inflate、onFinishInflate、requestLayout、onSizeChanged、invalidate、postInvalidate、setWillNotDraw、onAttachedToWindow、onDetachedFromWindow、ViewTreeObserver等</b>
* [自定义控件常用方法总结](http://www.jianshu.com/p/744550c02cf1)
* [如何学习自定义View有这些足够了 ](http://mp.weixin.qq.com/s/QxDfXZ4ydKGJHu_mFUPMTA)