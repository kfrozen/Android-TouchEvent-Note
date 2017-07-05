**Android中Touch事件的那些事**
	
	
首先是主要方法及调用顺序(MotionEvent.DOWN事件为起始事件，这里的顺序仅牵扯一个View本身，不牵扯分发)：
		
		dispatchTouchEvent(...) --> onInterceptTouchEvent(...) --> onTouchEvent(...)
	

接下来要知道的是，所有的touch事件都是先进行dispatch（由外层View向内层），当dispatch至最内层view时，再开始在onTouchEvent(...)中处理（由内层View向外层）

----------

		dispatchTouchEvent(...)
		{	
			/**代表已消费了该事件，此次及后续touch事件不会再继续向下传递，事件会被截留至此，后续的touch事件例如MOVE会继续传递至此**/
			return true;

			/**代表不消费该事件，但也不继续向下传递事件，事件将会被从View交回给Activity，后续的touch事件例如MOVE也不会再传递至此**/
			return false;

			/**不论返回true或false，事件都不会继续其在该ViewGroup中的传递，即View自身的onInterceptTouchEvent和onTouchEvent方法不会被调用，其子View不会接受到事件，其父View的onTouchEvent事件也不会被调用，true/false的区别仅在于DOWN之后的事件还会不会继续被给到这个方法中**/

			/**super方法中会调用该层View的onInterceptTouchEvent(...)事件以判断是否将touch事件拦截在该层，若为true，则该事件以及后续事件将被传入自身的onTouchEvent(...)事件而并不继续向下分发，若为false，则会向下分发至每一个子View**/
			return super.dispatchTouchEvent(...);
		}

----------

		onInterceptTouchEvent(...)
		{
			/**代表该View将拦截整次touch事件，事件将被传入自身的onTouchEvent(...)方法中并停止向其子View继续传递**/
			return true;

			/**代表该View将不会拦截此次touch事件，该事件及后续事件会继续向其子View传递（传递工作由dispatchEvent(...)完成）**/
			return false;


			/**对于绝大多数Mobile设备来说，等同于返回false，具体请参照源码**/
			return super.onInterceptTouchEvent(...);
		}

----------

		onTouchEvent(...)
		{
			/**代表该View已消费此次touch事件，事件将不会被传递给向上层(即其父View)的onTouchEvent()**/
			return true;

			/**代表该View不消费此次touch事件，事件将会被传递给向上层(即其父View)的onTouchEvent()**/
			return false;
		}

----------
		
		dispatchTouchEvent(...)       
		{									
			//Do something				
                                      										
			return false;
		}

		效果等同于：
		
		dispatchTouchEvent(...)
		{
			return super.dispatchTouchEvent();
		}
		+
		onInterceptTouchEvent(...)
		{
			return true;
		}
		+
		onTouchEvent(...)
		{
			//Do something
			
			return true；
		}


----------

下面说一下三个方法的返回值对彼此的影响，这里贴出一段ViewGroup关于dispatchTouchEvent(...)的源码

		@Override
    	public boolean dispatchTouchEvent(MotionEvent ev) 
		{
			......

			// Check for interception.
            final boolean intercepted;
            if (actionMasked == MotionEvent.ACTION_DOWN
                    || mFirstTouchTarget != null) {
                final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
                if (!disallowIntercept) {
                    intercepted = onInterceptTouchEvent(ev);
                    ev.setAction(action); // restore action in case it was changed
                } else {
                    intercepted = false;
                }
            } else {
                // There are no touch targets and this action is not an initial down
                // so this view group continues to intercept touches.
                intercepted = true;
            }

			......
		}

首先，任何类型的touch事件都会被首先传入dispatchTouchEvent(...)，该流程不受后续方法返回值的影响，因此如果需要在一个ViewGroup中针对所有touch事件对该ViewGroup做相应的事情，这里是不错的地方。

if (actionMasked == MotionEvent.ACTION_DOWN || mFirstTouchTarget != null) 从这一句代码可以看出，对于一个ViewGroup来说，想要进入其onInterceptTouchEvent(...)需要至少满足下面两个条件中的一个：

1. 此次事件是ACTION_DOWN；
2. 至少有一个子View的onTouchEvent(...)方法返回true，即消费了Touch事件。

若没有如2中所描述的这样的一个子View，且该ViewGroup本身也不消费Touch事件，则事件会被继续往上层抛，由其父ViewGroup(if any)进行处理，通常来说，如果该ViewGroup有另一个平级的兄弟ViewGroup，则事件会被传递至兄弟ViewGroup。

另一种情况，若依然没有如2中所描述的子View，但该ViewGroup消费了Touch事件，则事件不会继续向上抛，但后续的事件(例如ACTION_MOVE,ACTION_UP等)会直接经过dispatchTouchEvent(...)传入该ViewGroup的onTouchEvent(...)中，并不会进入onInterceptTouchEvent(...)，直到下次ACTION_DOWN发生。

两层View，上层的会挡住下层的事件，是因为上层的截获了onTouch，除非dispatchTouchEvent返回false，这样事件才会传到平级的下层中去。之前遇到的问题是因为这个导致的，最后解决办法是移开上层的view，不能用scrollBy，那样只能移动内容，必须用translation

