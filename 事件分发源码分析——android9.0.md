事件分发源码分析——android9.0

通过看书看博客几乎大家都知道了关于事件分发的基本的几个方法和几个结论，比如某个View拦截了down事件，那么所有的事件流都被它消耗，还有什么onTouch比onClick先判断等等，但是从来没有去真正分析最新的源码，今天就来根据最新的源码来从新学习事件分发，关键是**一步步探寻那些结论是怎么得出的**

首先activity的dispatchTouchEvent没得说

```java
 public boolean dispatchTouchEvent(MotionEvent ev) {
        if (ev.getAction() == MotionEvent.ACTION_DOWN) {
            onUserInteraction();
        }
        if (getWindow().superDispatchTouchEvent(ev)) {
            return true;
        }
        return onTouchEvent(ev);
    }
```

可以看到第二个判断是调用了window的superDispatchTouchEvent，如果点进去看的话会发现phoneWindow(window的唯一实现类)里面调用了decorView的DispatchTouchEvent，所以核心就是在ViewGroup中的DispatchTouchEvent



```java
  @Override
    public boolean dispatchTouchEvent(MotionEvent ev) {


        boolean handled = false;

        //过滤不合理的Event
        if (onFilterTouchEventForSecurity(ev)) {
            final int action = ev.getAction();
            final int actionMasked = action & MotionEvent.ACTION_MASK;

            // 处理最初的Down事件，如果是Down就重置触摸状态,
            // 可以理解为把disallowIntercept赋值为false，默认不拦截。
            if (actionMasked == MotionEvent.ACTION_DOWN) {
                cancelAndClearTouchTargets(ev);
                resetTouchState();
            }

            //检测是否拦截
            final boolean intercepted;
            //这里注意了，又两个重要的判断，只有当事件为Down或者 mFirstTouchTarget != null才会判断是否拦截
            //关键是这个mFirstTouchTarget 是什么东西？可以理解为上一个接受事件的View
            //mFirstTouchTarget是TouchTarget类的对象，是ViewGroup的一个成员变量，TouchTarget类是对View的一个封装，可以理解为接受touch事件的对象（其实就是View）
            //那么 mFirstTouchTarget在哪里被赋值呢？这里就先说了，是在后面找到并消费事件的View的时候被赋值
            if (actionMasked == MotionEvent.ACTION_DOWN
                    || mFirstTouchTarget != null) {
                //(mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0 这个位运算是否为0可以决定是否拦截
                //上面的重置状态的方法其实是改变了这两个变量
                final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
                if (!disallowIntercept) {
                    //不阻止拦截以后，进一步确认是否拦截，由onInterceptTouchEvent确认
                    intercepted = onInterceptTouchEvent(ev);
                    ev.setAction(action); // restore action in case it was changed
                } else {
                    intercepted = false;
                }
            } else {
               //这里既没有可以接受事件的view而且down不是初始的down,所以ViewGroup直接拦截了
                // There are no touch targets and this action is not an initial down
                // so this view group continues to intercept touches.
                intercepted = true;
            }
            //到此为止intercepted变量就确认了


            // 检测是否取消操作，一般是手指到屏幕外了
            final boolean canceled = resetCancelNextUpFlag(this)
                    || actionMasked == MotionEvent.ACTION_CANCEL;
            final boolean split = (mGroupFlags & FLAG_SPLIT_MOTION_EVENTS) != 0;
            TouchTarget newTouchTarget = null;
            boolean alreadyDispatchedToNewTouchTarget = false;


            //最终根据canceled和intercepted进行判断，如果既没有取消也没有拦截事件
            //为什么第一遍看源码不容易看懂，是因为代码不止会执行一次，会考虑多种情况。
            //注意，什么时候会执行下面这个if语句，首先是第一次Down的时候并且ViewGroup不拦截
            //那么什么时候不拦截呢？其实是上面那个位运算(mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0
            //如果仔细看上面决定intercepted变量的代码的话会发现如果(mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0，那么就不会执行下面的onInterceptTouchEvent(ev);来再次判断intercepted，而是intercepted直接为false,即不拦截，会执行下面的if,而我们熟知的requestDisallowInterceptTouchEvent就是改变了那个位运算的值，从而直接改变intercepted。同样的，当mFirstTouchTarget找到之后，也不会再询问onInterceptTouchEvent(ev)了。
            
            if (!canceled && !intercepted) {

                //如果是按下的话执行下面代码，但是如果不是Down是其他事件就不会执行下面if,然后你会发现就没有代码了，就是说如果不是Down这里就什么也不干。
                if (actionMasked == MotionEvent.ACTION_DOWN
                        || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN)
                        || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
                //这里是Down
                    
                    final int actionIndex = ev.getActionIndex(); // always 0 for down
                    final int idBitsToAssign = split ? 1 << ev.getPointerId(actionIndex)
                            : TouchTarget.ALL_POINTER_IDS;


                    final int childrenCount = mChildrenCount;
                    if (newTouchTarget == null && childrenCount != 0) {
                        final float x = ev.getX(actionIndex);
                        final float y = ev.getY(actionIndex);

                        //buildTouchDispatchChildList()就是把子View根据Z轴的值进行排列
                        //因为view是层层叠加在一起的
                        final ArrayList<View> preorderedList = buildTouchDispatchChildList();
                        final boolean customOrder = preorderedList == null
                                && isChildrenDrawingOrderEnabled();
                        //取出所有的子View
                        final View[] children = mChildren;
                        //由于刚刚排序了，所以倒着来
                        for (int i = childrenCount - 1; i >= 0; i--) {
                            final int childIndex = getAndVerifyPreorderedIndex(
                                    childrenCount, i, customOrder);
                            //取出View
                            final View child = getAndVerifyPreorderedView(
                                    preorderedList, children, childIndex);

                            //这里做了两个很重要的判断，根据判断的名字就推断为，是否该view可以接收到事件
                            //和是否该view在我们触摸的范围之内
                            //第一个判断其实就是如果view不可见或者处于动画状态就返回false,那么直接就continue不管了
                            //第二个判断是检测触碰事件是否在view的区域内，如果不在那就也continue不管了
                            if (!canViewReceivePointerEvents(child)
                                    || !isTransformedTouchPointInView(x, y, child, null)) {
                                ev.setTargetAccessibilityFocus(false);
                                continue;
                            }



                            //这里确定了view可以接受事件就从这里开始正式的事件分                                                       // 发,dispatchTransformedTouchEvent()里面其实是调用了view的                                              //dispatchTouchEvent(),也就是把事件传递给子View了，如果消费就返回ture
                            if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
                                // Child wants to receive touch within its bounds.
                               //这里的话事件确定被该View消费
                                mLastTouchDownTime = ev.getDownTime();
                                if (preorderedList != null) {
                                    // childIndex points into presorted list, find original index
                                    for (int j = 0; j < childrenCount; j++) {
                                        if (children[childIndex] == mChildren[j]) {
                                            mLastTouchDownIndex = j;
                                            break;
                                        }
                                    }
                                } else {
                                    mLastTouchDownIndex = childIndex;
                                }
                                mLastTouchDownX = ev.getX();
                                mLastTouchDownY = ev.getY();
                               //这里来了，在下面的addTouchTarget就把确定消费事件的View保存起来，就是说上面的一大段就是找到消费事件的View并消费事件并保存view为mFirstTarget,就是前面的那个变量。
                                newTouchTarget = addTouchTarget(child, idBitsToAssign);
                                alreadyDispatchedToNewTouchTarget = true;
                                break;
                            }

                            // The accessibility focus didn't handle the event, so clear
                            // the flag and do a normal dispatch to all children.
                            ev.setTargetAccessibilityFocus(false);
                        }
                        if (preorderedList != null) preorderedList.clear();
                    }   
         
                }
            }

            // Dispatch to touch targets.
            //到了这里，如果事件被消费了，那么mFirstTouchTarget一定不为Null,如果为Null，那么一定是没有找到可以消费事件的View
            if (mFirstTouchTarget == null) {
                // No touch targets so treat this as an ordinary view.
               //所以传入null,dispatchTransformedTouchEvent里面会判断如果是null的话就调用自己的onTouchEvent，ViewGroup自己消费事件
                handled = dispatchTransformedTouchEvent(ev, canceled, null,
                        TouchTarget.ALL_POINTER_IDS);
            } else {
               //如果是其他Down之后的事件，那么mFirstTouchTarget已经被第一次down的时候确认了（上面的循环检测判断只有是down的时候才会执行），那么这里就会直接交给mFirstTouchTarget里存的那个View来执行。也就是我们所说的事件流都交给这个View了，这个理就是这么来的
                // Dispatch to touch targets, excluding the new touch target if we already
                // dispatched to it.  Cancel touch targets if necessary.
                TouchTarget predecessor = null;
                TouchTarget target = mFirstTouchTarget;
                while (target != null) {
                    final TouchTarget next = target.next;
                    if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {
                        handled = true;
                    } else {
                        final boolean cancelChild = resetCancelNextUpFlag(target.child)
                                || intercepted;
                       //看，下面就是所有的事件流都给了这个第一次找到的target
                        if (dispatchTransformedTouchEvent(ev, cancelChild,
                                target.child, target.pointerIdBits)) {
                            handled = true;
                        }
                        if (cancelChild) {
                            if (predecessor == null) {
                                mFirstTouchTarget = next;
                            } else {
                                predecessor.next = next;
                            }
                            target.recycle();
                            target = next;
                            continue;
                        }
                    }
                    predecessor = target;
                    target = next;
                }
            }

            // Update list of touch targets for pointer up or cancel, if needed.
            if (canceled
                    || actionMasked == MotionEvent.ACTION_UP
                    || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
                resetTouchState();
            } else if (split && actionMasked == MotionEvent.ACTION_POINTER_UP) {
                final int actionIndex = ev.getActionIndex();
                final int idBitsToRemove = 1 << ev.getPointerId(actionIndex);
                removePointersFromTouchTargets(idBitsToRemove);
            }
        }

        if (!handled && mInputEventConsistencyVerifier != null) {
            mInputEventConsistencyVerifier.onUnhandledEvent(ev, 1);
        }
        return handled;
    }

```

到此为止，基本上ViewGroup的都搞清楚了，但是传递到View里面的事件还没有分析，所以看看View里面的事件分析

```java
    public boolean dispatchTouchEvent(MotionEvent event) {

//首先还是一个结果，表示是否消耗了事件，默认是false
        boolean result = false;


        if (onFilterTouchEventForSecurity(event)) {
            if ((mViewFlags & ENABLED_MASK) == ENABLED && handleScrollBarDragging(event)) {
                result = true;
            }
            //noinspection SimplifiableIfStatement
            ListenerInfo li = mListenerInfo;
           //这里就关键了，看下面这个判断，可以看到当TouchListener存在并且它的ontouch方法返回ture的话，result就直接为ture了，并且下面的判断也不会进行了。而onTouchEvent方法在下面才进行，这就是为什么如果onTouch返回了ture后，cilick事件就不被执行了（因为cilick方法在onTouchEvent方里面）。因为事件被onTouch消费了！
            if (li != null && li.mOnTouchListener != null
                    && (mViewFlags & ENABLED_MASK) == ENABLED
                    && li.mOnTouchListener.onTouch(this, event)) {
                result = true;
            }

            if (!result && onTouchEvent(event)) {
                result = true;
            }
        }

        if (!result && mInputEventConsistencyVerifier != null) {
            mInputEventConsistencyVerifier.onUnhandledEvent(event, 0);
        }

        // Clean up after nested scrolls if this is the end of a gesture;
        // also cancel it if we tried an ACTION_DOWN but we didn't want the rest
        // of the gesture.
        if (actionMasked == MotionEvent.ACTION_UP ||
                actionMasked == MotionEvent.ACTION_CANCEL ||
                (actionMasked == MotionEvent.ACTION_DOWN && !result)) {
            stopNestedScroll();
        }

        return result;
    }
```

然后View的onTouchEvent就没必要看了，就是swich分支处理事件。

到此就把这一个流程都理清楚了。



