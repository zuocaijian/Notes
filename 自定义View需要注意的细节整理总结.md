## 一、View触摸滚动事件计算方向总结
 **当手指向下滑动**  
  1. 内容向上滚动。在源码中一般使用scrollUp来描述；  
  2. yDiff = y - lastY 为正值；  
  3. translationY、 offsetBottomAndTop()以及getY() 为正值；   
  4. scrollBy()、 scrollTo()以及getScroll()为负值；  
  5. direction为-1； (是指向View.canScrollVertically(int direction) 方法中传递的参数);  
  6. 通过VelocityTracker获得的VelocityY为正值。 但是使用Scroller.fling()方法传递参数时，需要填入VelocityY的相反数。  

## 二、View测绘等回调方法的顺序  
  **onFinishInflate()** --> **onAttachedToWindow()** --> **onMeasure(int, int)** --> **onSizeChanged(int, int, int, int)** --> **onLayout(boolean, int, int, int, int)** --> **onMeasure(int, int)** --> **onLayout(boolean, int, int, int, int)** --> ... --> **onDetachedFromWindow()**