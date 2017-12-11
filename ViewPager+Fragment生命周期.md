> 当Fragment和ViewPager配合使用时，由于ViewPager的缓存机制，使得Fragment的生命周期回掉函数失去了其原来的意义。再实际开发中，每一页Fragment都会需要加载大量的数据，如果不做任何处理，那么缓存机制会使得在第一次打开Activity时同时加载多页Fragment的数据，导致界面卡顿，浪费用户流量。因此我们需要合理控制数据的加载，正确利用Fragment的生命周期回掉函数。

### ViewPager + Fragment 的生命周期回掉

在Activity中放置一个ViewPager控件，并且在ViewPager中添加3个及以上Fragment，打开Activity，我们发现Fragment的生命周期回掉函数顺序如下：

	1. SimpleFragment0: setUserVisibleHint() -> false  
	2. SimpleFragment1: setUserVisibleHint() -> false  
	3. SimpleFragment0: setUserVisibleHint() -> true  
	4. SimpleFragment0: onCreate()  
	5. SimpleFragment1: onCreate()  
	6. SimpleFragment0: onCreateView()  
	7. SimpleFragment0: onViewCreated()  
	8. SimpleFragment0: onActivityCreated()  
	9. SimpleFragment0: onStart()  
	10. SimpleFragment0: onResume()  
	11. SimpleFragment1: onCreateView()  
	12. SimpleFragment1: onViewCreated()  
	13. SimpleFragment1: onStart()
	14. SimpleFragment1: onResume()

关闭Activity，生命周期回掉函数顺序如下：
	1. SimpleFragment0: onPause()  
	2. SimpleFragment1: onPause()  
	3. SimpleFragment0: onStop()  
	4. SimpleFragment1: onStop()  
	5. SimpleFragment0: onDestroy()  
	6. SimpleFragment1: onDestroy()  

以上生命周期中比较不合理的地方：
  - 在第一次打开Activity时，setUserVisibleHint()方法首先被调用，并且被调用两次，先是false，然后是true。而此时onCreateView()并未被调用，也即View尚未被创建。  
  - 在第一次打开Activity时，当前显示且可交互的是第一个fragment，而第二个fragment已经被创建，并且被onResume()了。  
针对以上情况，我们需要对Fragment略作处理，实现懒加载，即在fragment需要显示给用户的时候才去加载数据。具体代码如下：  

		public abstract class BaseLazyFragment extends BaseFragment {
		
		    protected boolean isFragmentVisible;
		    protected boolean hasCreateView;
		
		    protected boolean hasVisibleChanged;
		
		    protected boolean isFirstVisible = true;
		
		    @Override
		    public void onCreate(@Nullable Bundle savedInstanceState) {
		        super.onCreate(savedInstanceState);
		        isFragmentVisible = false;
		        hasCreateView = false;
		    }
		
		    @Override
		    public void onViewCreated(View view, @Nullable Bundle savedInstanceState) {
		        super.onViewCreated(view, savedInstanceState);
		        if (!hasCreateView && getUserVisibleHint()) {//说明fragment是第一次加载
		            isFragmentVisible = true;
		            onVisibleChanged(true);
		        }
		    }
		
		    @Override
		    public void onResume() {
		        super.onResume();
		
		        if (isFragmentVisible && !hasVisibleChanged) {
		            onVisibleChanged(true);
		        }
		    }
		
		    @Override
		    public void onPause() {
		        super.onPause();
		        hasVisibleChanged = false;
		    }
		
		    @Override
		    public void setUserVisibleHint(boolean isVisibleToUser) {
		        super.setUserVisibleHint(isVisibleToUser);
		        if (null == mView) {
		            return;
		        }
		        hasCreateView = true;
		        if (isVisibleToUser) {
		            isFragmentVisible = true;
		            onVisibleChanged(true);
		            return;
		        }
		        if (isFragmentVisible) {
		            isFragmentVisible = false;
		            onVisibleChanged(false);
		        }
		    }
		
		    protected abstract void onFragmentVisibleChanged(boolean isVisible);
		
		    private void onVisibleChanged(boolean isVisible) {
		        hasVisibleChanged = true;
		        onFragmentVisibleChanged(isVisible);
		        if (isVisible) {
		            isFirstVisible = false;
		        }
		    }
		}

在使用时只需继承BaseLazyFragment类，然后覆写onFragmentVisibleChanged(boolean isVisible)方法，根据**isVisible**判断当前Fragment时候对用户可见，做数据的加载与释放。同时，可以根据**isFirstVisible**来判断当前Fragment是不是首次加载。另外，当应用从前台切换到后台、或者从后台切换到前台，也都会触发onFragmentVisibleChanged(boolean isVisible)方法，从而正确的处理数据加载逻辑。