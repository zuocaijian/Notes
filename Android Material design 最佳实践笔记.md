# 一、NestedScrollView 嵌套 RecyclerView  
* 问题描述： RecyclerView抢占焦点，导致进入页面时直接滚动到Recyclerview顶部  
解决方法：在NestedScrollView的直接子View中添加属性：  

    	android:focusable="true"  
    	android:focusableInTouchMode="true"

最终xml如下：  

	<android.support.v4.widget.NestedScrollView xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/nsv"
    	android:layout_width="match_parent"
    	android:layout_height="match_parent"
    	android:orientation="vertical">

		<LinearLayout
			android:layout_width="match_parent"
        	android:layout_height="match_parent"
        	android:focusable="true"
        	android:focusableInTouchMode="true"
        	android:orientation="vertical">

			...
			...
			<android.support.v7.widget.RecyclerView
				android:id="@+id/rv"
            	android:layout_width="match_parent"
            	android:layout_height="match_parent" />

		</LinearLayout>
	
	</android.support.v4.widget.NestedScrollView>  

* 问题描述：滚动时，滑动不流畅  
解决办法：对Adapter和RecylerView分别设置如下属性：  

		mLayoutManager.setSmoothScrollbarEnabled(true);
        mLayoutManager.setAutoMeasureEnabled(true);
        mRv.setNestedScrollingEnabled(false);

---
# 二、使用Material Design提供的沉浸式主题

* 使用xml设置主题  
	1. 在res/values/styles.xml中新建如下主题
  
			<style name="MD_Theme" parent="AppTheme">
        		<item name="windowActionBar">false</item>
        		<item name="android:windowNoTitle">true</item>
    		</style>

	2. 同时在res/values-v21/styles.xml中新建同名主题  
	
			<style name="MD_Theme" parent="AppTheme">
        		<item name="windowActionBar">false</item>
        		<item name="windowNoTitle">true</item>
    		</style>
	3. 在AndroidManifest文件中将主题设置在application标签下或activity标签下即可  
<p>  
* 使用ToolBar代替ActionBar  
	1. 在Activity的布局中文件中添加ToolBar的引用  
 
			<android.support.v7.widget.Toolbar
            	android:id="@+id/tb"
            	android:layout_width="match_parent"
            	android:layout_height="wrap_content"/>

	2. 在Activity中通过代码替换ActionBar  

        	mToolBar.setLogo(R.mipmap.ic_launcher);  
        	mToolBar.setTitle("仿知乎首页");  
        	mToolBar.setSubtitle("md");  
        	setSupportActionBar(mToolBar);  
        	mToolBar.setNavigationIcon(R.mipmap.ic_launcher);  
        	mToolBar.setOnMenuItemClickListener(listener);

注意：  

- logo、title等要在setSupportActionBar之前设置；  
- setNavigationIcon、setOnMenuItemClickListener要在setSupportActionBar之后设置。

# 三、CoordinatorLayout、AppBarLayout、CollapsingToolbarLayout、FloatingButtonAction、Behavior联用，仿知乎首页效果

> 参考CSDN严振杰Material design系列博客[Material Design系列，自定义Behavior支持所有View](http://blog.csdn.net/yanzhenjie1003/article/details/52205665)

* 关于AppBarLayout，一个非常重要的属性是**app:layout_scrollFlags**，他可以定制当某个可滚动的控件（NestedScrollView，RecyclerView等）滚动手势发生变化时，其内部的子View实现何种变化（一起滚动、固定...）。该属性的值有以下几种：  
	1. scroll:值设为scroll的View会跟随滚动事件一起发生移动;  
	2. enterAlways:值设为enterAlways的View,当ScrollView往下滚动时，该View会直接往下滚动。而不用考虑ScrollView是否在滚动;  
	3. exitUntilCollapsed：值设为exitUntilCollapsed的View，当这个View要往上逐渐“消逝”时，会一直往上滑动，直到剩下的的高度达到它的最小高度后，再响应ScrollView的内部滑动事件。使用该值时需要同时设置该View的最小高度android：minHieght属性；
	4. enterAlwaysCollapsed：是enterAlways的附加选项，一般跟enterAlways一起使用，它是指，View在往下“出现”的时候，首先是enterAlways效果，当View的高度达到最小高度时，View就暂时不去往下滚动，直到ScrollView滑动到顶部不再滑动时，View再继续往下滑动，直到滑到View的顶部结束。设置该值时同样需要同时设置View的最小高度属性。  
> **在使用时AppBarLayout的app：layout_scrollFlags属性时，需要在可滚动的View中指定Behavior，即配置app:layout_behavior="@string/appbar_scrolling_view_behavior"**  

* 关于CollapsingToolbarLayout，用于包装Toolbar，且需要作为AppBarLayout的直接子View。该ViewGroup向子View提供了app:layout_collapseMode等属性来实现以下几种效果：
	1. 折叠Title（Collapsing title）：当布局内容全部显示出来时，title是最大的，但是随着View逐步移出屏幕顶部，title变得越来越小。你可以通过调用setTitle函数来设置title；
	2. 内容纱布（Content scrim）：根据滚动的位置是否到达一个阀值，来决定是否对View“盖上纱布”。可以通过setContentScrim(Drawable)来设置纱布的图片；
	3. 状态栏纱布（Status bar scrim)：根据滚动位置是否到达一个阀值决定是否对状态栏“盖上纱布”，你可以通过setStatusBarScrim(Drawable)来设置纱布图片，但是只能在LOLLIPOP设备上面有作用；
	4. 视差滚动子View(Parallax scrolling children):子View可以选择在当前的布局当时是否以“视差”的方式来跟随滚动。（PS:其实就是让这个View的滚动的速度比其他正常滚动的View速度稍微慢一点）。将布局参数app:layout_collapseMode设为parallax；
	5. 将子View位置固定(Pinned position children)：子View可以选择是否在全局空间上固定位置，这对于Toolbar来说非常有用，因为当布局在移动时，可以将Toolbar固定位置而不受移动的影响。 将app:layout_collapseMode设为pin。  
以上关于AppBarLayout和CoolapsingToolbarLayout的典型xml布局如下所示：  

			<?xml version="1.0" encoding="utf-8"?>
			<android.support.design.widget.CoordinatorLayout xmlns:android="http://schemas.android.com/apk/res/android"
			    xmlns:app="http://schemas.android.com/apk/res-auto"
			    android:layout_width="match_parent"
			    android:layout_height="match_parent" >
	
			    <android.support.design.widget.AppBarLayout
			        android:layout_width="match_parent"
			        android:layout_height="wrap_content"
			        android:theme="@style/ThemeOverlay.AppCompat.Dark.ActionBar">
			
			        <android.support.design.widget.CollapsingToolbarLayout
			            android:layout_width="match_parent"
			            android:layout_height="wrap_content"
			            app:expandedTitleMarginEnd="64dp"
			            app:expandedTitleMarginStart="48dp"
			            app:layout_scrollFlags="scroll|exitUntilCollapsed">
			
			            <ImageView
			                android:id="@+id/main.backdrop"
			                android:layout_width="wrap_content"
			                android:layout_height="300dp"
			                android:scaleType="centerCrop"
			                android:src="@drawable/material_img"
			                app:layout_collapseMode="parallax" />
			
			            <android.support.v7.widget.Toolbar
			                android:id="@+id/toolbar"
			                android:layout_width="match_parent"
			                android:layout_height="?android:attr/actionBarSize"
			                app:layout_collapseMode="pin"  />
			
			        </android.support.design.widget.CollapsingToolbarLayout>
			    </android.support.design.widget.AppBarLayout>
			
			    <android.support.v4.widget.NestedScrollView
			
			        android:layout_width="match_parent"
			        android:layout_height="wrap_content"
			        android:paddingTop="50dp"
			        app:layout_behavior="@string/appbar_scrolling_view_behavior">
			
			        <TextView
			            android:layout_width="match_parent"
			            android:layout_height="wrap_content"
			            android:text="@string/my_txt"
			            android:textSize="20sp" />
			
			    </android.support.v4.widget.NestedScrollView>
	
			</android.support.design.widget.CoordinatorLayout>

> 如果你希望拖动过程中状态栏是透明的，可以在CollapsingToolbarLayout中加入 app:statusBarScrim="@android:color/transparent"，并且在onCreate中调用getWindow().addFlags(WindowManager.LayoutParams.FLAG_TRANSLUCENT_STATUS)将状态栏设置为透明即可。  
> 以上参考[玩转AppBarLayout，更酷炫的顶部栏](http://www.jianshu.com/p/d159f0176576)

* 关于FloatingActionButton，影响效果的属性主要有：  
	1.  app:borderWidth=""----------------------边框宽度，通常设置为0 ，用于解决Android 5.X设备上阴影无法正常显示的问题  
	2.  app:backgroundTint=""------------------按钮的背景颜色，不设置，默认使用theme中colorAccent的颜色  
	3.  app:rippleColor=""------------------------点击的边缘阴影颜色  
	4.  app:elevation=""--------------------------边缘阴影的宽度  
	5.  app:pressedTranslationZ="16dp"-----点击按钮时，按钮边缘阴影的宽度，通常设置比elevation的数值大
> **如果不设置FloatingActionButton的点击事件，则不会显示点击效果**

### 综合使用上述控件，并自定义一个Behavior，最终实现仿知乎首页效果:  
![效果图](http://img.027cgb.cn/20170713/20177135001775731906.gif)  
关键代码如下：  
* **xml布局** 
	
	<?xml version="1.0" encoding="utf-8"?>
	<android.support.design.widget.CoordinatorLayout xmlns:android="http://schemas.android.com/apk/res/android"
	    xmlns:app="http://schemas.android.com/apk/res-auto"
	    android:layout_width="match_parent"
	    android:layout_height="match_parent">

	    <android.support.design.widget.AppBarLayout
	        android:id="@+id/abl"
	        android:layout_width="match_parent"
	        android:layout_height="?attr/actionBarSize">
	
	        <android.support.v7.widget.Toolbar
	            android:id="@+id/tb"
	            android:layout_width="match_parent"
	            android:layout_height="wrap_content"
	            app:layout_scrollFlags="enterAlways|scroll|snap">
	
	        </android.support.v7.widget.Toolbar>
	    </android.support.design.widget.AppBarLayout>
	
	    <android.support.v7.widget.RecyclerView
	        android:id="@+id/rv"
	        android:layout_width="match_parent"
	        android:layout_height="match_parent"
	        android:background="@color/colorAccent"
	        app:layout_behavior="@string/appbar_scrolling_view_behavior" />
	
	    <LinearLayout
	        android:id="@+id/ll_tab"
	        android:layout_width="match_parent"
	        android:layout_height="wrap_content"
	        android:background="#d88"
	        android:orientation="horizontal"
	        app:layout_behavior="@string/bottom_sheet_behavior">
	
	        <Button
	            android:layout_width="0dp"
	            android:layout_height="wrap_content"
	            android:layout_weight="1"
	            android:padding="10dp"
	            android:text="第一个" />
	
	        <Button
	            android:layout_width="0dp"
	            android:layout_height="wrap_content"
	            android:layout_weight="1"
	            android:padding="10dp"
	            android:text="第二个" />
	
	        <Button
	            android:layout_width="0dp"
	            android:layout_height="wrap_content"
	            android:layout_weight="1"
	            android:padding="10dp"
	            android:text="第三个" />
	
	        <Button
	            android:layout_width="0dp"
	            android:layout_height="wrap_content"
	            android:layout_weight="1"
	            android:padding="10dp"
	            android:text="第四个" />
	    </LinearLayout>
	
	    <android.support.design.widget.FloatingActionButton
	        android:id="@+id/fab"
	        android:layout_width="wrap_content"
	        android:layout_height="wrap_content"
	        android:layout_marginBottom="60dp"
	        android:layout_marginRight="15dp"
	        android:src="@mipmap/ic_launcher"
	        app:borderWidth="0dp"
	        app:elevation="8dp"
	        app:layout_anchor="@id/rv"
	        app:layout_anchorGravity="bottom|right"
	        app:layout_behavior="com.zcj.test.myview.widget.custom.ScaleDownBehavior" />

	</android.support.design.widget.CoordinatorLayout>
* **自定义Behavior：ScaleDownBehavior，上滑滚动控件时FloatingActionButton消失，下滑滚动控件时FloatingActionButton出现**  
	
		public class ScaleDownBehavior extends FloatingActionButton.Behavior {
		
		    private static final String TAG = ScaleDownBehavior.class.getName();
		
		    public ScaleDownBehavior() {
		        super();
		    }
		
		    public ScaleDownBehavior(Context context, AttributeSet set) {
		        super(context, set);
		    }
		
		    @Override
		    public boolean onStartNestedScroll(CoordinatorLayout coordinatorLayout, FloatingActionButton child, View directTargetChild, View target, int nestedScrollAxes) {
		        if (nestedScrollAxes == ViewCompat.SCROLL_AXIS_VERTICAL) {
		            return true;
		        }
		        return super.onStartNestedScroll(coordinatorLayout, child, directTargetChild, target, nestedScrollAxes);
		    }
		
		    @Override
		    public void onNestedScroll(CoordinatorLayout coordinatorLayout, FloatingActionButton child, View target, int dxConsumed, int dyConsumed, int dxUnconsumed, int dyUnconsumed) {
		        if (dyConsumed > 0 || dyUnconsumed > 0) {
		            //手指上滑，隐藏
		            child.clearAnimation();
		            AnimUtil.hide(child, null);
		
		            BottomSheetBehavior behavior = BottomSheetBehavior.from(coordinatorLayout.findViewById(R.id.ll_tab));
		            behavior.setPeekHeight(0);
		            behavior.setState(BottomSheetBehavior.STATE_COLLAPSED);
		
		        } else if (dyConsumed < 0 || dyUnconsumed < 0) {
		            //手指下滑，显示
		            child.clearAnimation();
		            AnimUtil.show(child, null);
		
		            BottomSheetBehavior behavior = BottomSheetBehavior.from(coordinatorLayout.findViewById(R.id.ll_tab));
		            behavior.setPeekHeight(0);
		            behavior.setState(BottomSheetBehavior.STATE_EXPANDED);
		        }
		    }
		
		    private static class AnimUtil {
		        public static void hide(View view, ViewPropertyAnimatorListener listener) {
		            ViewCompat.animate(view)
		                    .scaleX(0.7f)
		                    .scaleY(0.7f)
		                    .alpha(0f)
		                    .setDuration(300)
		                    .setListener(listener)
		                    .start();
		        }
		
		        public static void show(View view, ViewPropertyAnimatorListener listener) {
		            ViewCompat.animate(view)
		                    .scaleX(1f)
		                    .scaleY(1f)
		                    .alpha(1f)
		                    .setDuration(300)
		                    .setListener(listener)
		                    .start();
		        }
		    }
		}
 

---
# NestedScrollChild、NestedScrollParent、Behavior方法回调源码跟踪  

以startNestedScroll与onStartNestedScroll为例分析

1. 在NestedScrollChild的实现类中：

			public boolean onTouchEvent(MotionEvent e) {
				...
				switch (action) {
            		case MotionEvent.ACTION_DOWN: {
						...
						//nestedScrollAxis为方向，分为上下放下、左右方向
						startNestedScroll(nestedScrollAxis);
						...
					}
					...
				}
			}  


2. startNestedScroll(nestedScrollAxis)方法只有一句代码：  

		getScrollingChildHelper().startNestedScroll(axes)

getScrollingChildHelper()返回一个NestedScrollingChildHelper实例，并将该实现了NestedScrollChild接口的实例作为其构造参数  
3. 在NestedScrollingChildHelper中：  

	while (p != null) {
        if (ViewParentCompat.onStartNestedScroll(p, child, mView, axes)) {
            mNestedScrollingParent = p;
            ViewParentCompat.onNestedScrollAccepted(p, child, mView, axes);
            return true;
		}
		if (p instanceof View) {
            child = (View) p;
        }
        p = p.getParent();
	}

其中，p=mView.getParent()，而mView由NestedScrollingChildHelper的构造函数中传递进来，一般为RecyclerView、NestedScrollView等控件，并且child即为mView  
4. ViewParentCompat采用了桥接/代理模式，具体方法由IMPL的子类同名方法实现，下面以API21为例，其onStartNestedScroll方法中只有一行代码：  

	public boolean onStartNestedScroll(ViewParent parent, View child, View target,
                int nestedScrollAxes) {
		return parent.onStartNestedScroll(child, target, nestedScrollAxes);
	}

通过以上代码，就会回调实现了NestedScrollParent接口的ViewGroup中的onStartNestedScroll，通常是我们用的CoordinatorLayout。  
5. 在实现了NestedScrollParent接口的ViewGroup中的onStartNestedScroll方法又会调用实现了NestedScrollChild接口的onStartNestedScroll方法，如下：  

		public boolean onStartNestedScroll(View child, View target, int nestedScrollAxes) {
			...
			final LayoutParams lp = (LayoutParams) view.getLayoutParams();
        	final Behavior viewBehavior = lp.getBehavior();
        	if (viewBehavior != null) {
        	    final boolean accepted = viewBehavior.onStartNestedScroll(this, view, child, target,
        	            nestedScrollAxes);
        	    handled |= accepted;
	
        	    lp.acceptNestedScroll(accepted);
        	} else {
        	    lp.acceptNestedScroll(false);
        	}
			...
		}  

可以看出，此处调用了Behavior的onStartNestedScroll方法。至此，整个回调方法链就进行完了。  
6. **总结**：以上，可以看出在Behavior的onStartNestedScroll方法中，五个参数依次为：  

- 实现了NestedScrollParent接口的ViewGroup，通常是CoordinatorLayout；  
- 使用该Behavior的view；	
- 消费了onStartNestedScroll的ViewGroup(实现了NestedScrollParent接口，也即是参数1)的子View，也是实现了NestedScrollChild的控件或上N级父控件，某些情况下等于参数4；  
- 实现了NestedScrollChild的控件，该控件是嵌套滑动系列事件的发起者；  
- 滑动方向  
