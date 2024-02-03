#### dispatchTouchEvent@ViewGroup

事件分发

```java
----> dispatchTouchEvent@ViewGroup:

public boolean dispatchTouchEvent(MotionEvent ev) {
    ...
		
    //handled：当前事件是否有view消费
    boolean handled = false;
  	//onFilterTouchEventForSecurity：过滤一下模糊的事件？？？什么鬼
    if (onFilterTouchEventForSecurity(ev)) {
      	
      	//获得当前事件的类型
        final int action = ev.getAction();
        final int actionMasked = action & MotionEvent.ACTION_MASK;

        // 事件为DOWN事件时，说明是个新的事件序列，状态清理下
        if (actionMasked == MotionEvent.ACTION_DOWN) {
						//清除上一个事件序列的TouchTarget链表
            cancelAndClearTouchTargets(ev);
          	//清除FLAG_DISALLOW_INTERCEPT等状态信息
            resetTouchState();
        }

        // 父View是否拦截判断
        final boolean intercepted;
        if (actionMasked == MotionEvent.ACTION_DOWN
                || mFirstTouchTarget != null) {
          	//mFirstTouchTarget != null说明事件有子View消费
          	//所有在事件为down事件，或者事件有View消费时才会进来
          
          	//判断下是否不允许拦截事件
            final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
            if (!disallowIntercept) {
              	//允许拦截时，调用onInterceptTouchEvent判断是否需要拦截该事件
                intercepted = onInterceptTouchEvent(ev);
                ev.setAction(action); 
            } else {
              	//不允许拦截
                intercepted = false;
            }
        } else {
          	//不是down或者没人处理事件，父View直接拦截
            intercepted = true;
        }

       	...

        // 是否为取消事件
        final boolean canceled = resetCancelNextUpFlag(this)
                || actionMasked == MotionEvent.ACTION_CANCEL;

       	//isMouseEvent鼠标事件？？？
        final boolean isMouseEvent = ev.getSource() == InputDevice.SOURCE_MOUSE;
      	//split是否支持多指，true支持，两根手指按在不同的列表上同时滑动
        final boolean split = (mGroupFlags & FLAG_SPLIT_MOTION_EVENTS) != 0 && !isMouseEvent;
      	
        TouchTarget newTouchTarget = null;
        boolean alreadyDispatchedToNewTouchTarget = false;
      	
      	//不是取消事件，父view不拦截时，进行分发给子view
        if (!canceled && !intercepted) {
            ...
						
            //down或者多指down时才会分发
            if (actionMasked == MotionEvent.ACTION_DOWN
                    || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN)
                    || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
              
              	//actionIndex：第几根手指按下，down时为0。所以是（0，1，2...）
                final int actionIndex = ev.getActionIndex(); // always 0 for down
              	//记录下手指的id（1，10，100，1000...）
              	//idBitsToAssign：当前按下的手指id
                final int idBitsToAssign = split ? 1 << ev.getPointerId(actionIndex)
                        : TouchTarget.ALL_POINTER_IDS;

                //清除之前的记录
                removePointersFromTouchTargets(idBitsToAssign);

                final int childrenCount = mChildrenCount;
                if (newTouchTarget == null && childrenCount != 0) {
                  	//得到手指按下的位置x，y
                    final float x = ev.getXDispatchLocation(actionIndex);
                    final float y = ev.getYDispatchLocation(actionIndex);
                    //子view排序
                    final ArrayList<View> preorderedList = buildTouchDispatchChildList();
                  	
                    final boolean customOrder = preorderedList == null
                            && isChildrenDrawingOrderEnabled();
                    final View[] children = mChildren;
                    for (int i = childrenCount - 1; i >= 0; i--) {
                      	//从后往前或者按照z轴从大到小依次得到子View
                        final int childIndex = getAndVerifyPreorderedIndex(
                                childrenCount, i, customOrder);
                        final View child = getAndVerifyPreorderedView(
                                preorderedList, children, childIndex);

                        ...
                        //view不可见 或者 触摸坐标不在view的范围内，跳过
                        if (!child.canReceivePointerEvents()
                                || !isTransformedTouchPointInView(x, y, child, null)) {
                            ev.setTargetAccessibilityFocus(false);
                            continue;
                        }
												
                      	//单指是newTouchTarget == null
                        newTouchTarget = getTouchTarget(child);
                        if (newTouchTarget != null) {
                            //多指触摸到相同的child
                            newTouchTarget.pointerIdBits |= idBitsToAssign;
                            break;
                        }

                        resetCancelNextUpFlag(child);
                      	//dispatchTransformedTouchEvent子view分发
                        if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
                            //有子View消费了事件
                            
                          	//记录事件的相关信息，事件，index和位置x，y
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
                            mLastTouchDownX = x;
                            mLastTouchDownY = y;
                          	//前插到mFirstTouchTarget
                            newTouchTarget = addTouchTarget(child, idBitsToAssign);
                          	//记录下，分发成功
                            alreadyDispatchedToNewTouchTarget = true;
                            break;
                        }
                        ...
                    }
                    if (preorderedList != null) preorderedList.clear();
                }
								
                if (newTouchTarget == null && mFirstTouchTarget != null) {
                    //没有子View接收事件，将idBitsToAssign添加到链表最后的TouchTarget中
                    newTouchTarget = mFirstTouchTarget;
                    while (newTouchTarget.next != null) {
                        newTouchTarget = newTouchTarget.next;
                    }
                    newTouchTarget.pointerIdBits |= idBitsToAssign;
                }
            }
        }

        if (mFirstTouchTarget == null) {
            // 没有子View接收事件
            handled = dispatchTransformedTouchEvent(ev, canceled, null,
                    TouchTarget.ALL_POINTER_IDS);
        } else {
            //
            TouchTarget predecessor = null;
            TouchTarget target = mFirstTouchTarget;
            while (target != null) {
                final TouchTarget next = target.next;
              	//已经分发成功，直接返回true
                if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {
                    handled = true;
                } else {
                    final boolean cancelChild = resetCancelNextUpFlag(target.child)
                            || intercepted;
                  	//父View此时不拦截，事件交给分发给同一个view - target.child
                  	//父View此时拦截，将事件设置为CANCEL传递给子View，并清空mFirstTouchTarget链表
                  	//之后的事件会直接自己处理
                    if (dispatchTransformedTouchEvent(ev, cancelChild,
                            target.child, target.pointerIdBits)) {
                        handled = true;
                    }
                  	//取消或者开始拦截
                    if (cancelChild) {
                      	//清空链表，mFirstTouchTarget = null
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
          	//取消重置状态
            resetTouchState();
        } else if (split && actionMasked == MotionEvent.ACTION_POINTER_UP) {
          	//手指抬起，清空TouchTarget中的手指id
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

##### buildOrderedChildList@ViewGroup

```java
----> buildOrderedChildList@ViewGroup:

//getZ() = getElevation() + getTranslationZ();
//根据elevation和translationZ排序
ArrayList<View> buildOrderedChildList() {
    final int childrenCount = mChildrenCount;
  	//子View不多于1，或者所有子view的getZ()都是0，不排序null
    if (childrenCount <= 1 || !hasChildWithZ()) return null;

  	//容器初始化
    if (mPreSortedChildren == null) {
        mPreSortedChildren = new ArrayList<>(childrenCount);
    } else {
        mPreSortedChildren.clear();
        mPreSortedChildren.ensureCapacity(childrenCount);
    }

    final boolean customOrder = isChildrenDrawingOrderEnabled();
    for (int i = 0; i < childrenCount; i++) {
        final int childIndex = getAndVerifyPreorderedIndex(childrenCount, i, customOrder);
        final View nextChild = mChildren[childIndex];
        final float currentZ = nextChild.getZ();

        //根据z轴，从小到大的顺序排序
      	//z轴相等时，xml布局后面的view在后面
        int insertIndex = i;
        while (insertIndex > 0 && mPreSortedChildren.get(insertIndex - 1).getZ() > currentZ) {
            insertIndex--;
        }
        mPreSortedChildren.add(insertIndex, nextChild);
    }
    return mPreSortedChildren;
}
```

##### dispatchTransformedTouchEvent@ViewGroup

```java
----> dispatchTransformedTouchEvent@ViewGroup:

private boolean dispatchTransformedTouchEvent(MotionEvent event, boolean cancel,
        View child, int desiredPointerIdBits) {
    final boolean handled;

    //处理取消事件（父View拦截当前事件）
    final int oldAction = event.getAction();
    if (cancel || oldAction == MotionEvent.ACTION_CANCEL) {
      	//设置当前事件为ACTION_CANCEL
        event.setAction(MotionEvent.ACTION_CANCEL);
      	//自己处理或者将ACTION_CANCEL事件分发给子View
        if (child == null) {
            handled = super.dispatchTouchEvent(event);
        } else {
            handled = child.dispatchTouchEvent(event);
        }
        event.setAction(oldAction);
        return handled;
    }

  	//多指处理
    final int oldPointerIdBits = event.getPointerIdBits();
    final int newPointerIdBits = oldPointerIdBits & desiredPointerIdBits;
		
    if (newPointerIdBits == 0) {
        return false;
    }
  	
  	//MotionEvent转换
  	//事件在parent和子view中的偏移不一定一样
  	//多指时，在parent中actionMask是pointer_down，而在子view中不一定，可能是action_down（两根手指按在不同的view上）
    final MotionEvent transformedEvent;
    if (newPointerIdBits == oldPointerIdBits) {
        if (child == null || child.hasIdentityMatrix()) {
            if (child == null) {
                handled = super.dispatchTouchEvent(event);
            } else {
              	//偏移改变
                final float offsetX = mScrollX - child.mLeft;
                final float offsetY = mScrollY - child.mTop;
                event.offsetLocation(offsetX, offsetY);

                handled = child.dispatchTouchEvent(event);

                event.offsetLocation(-offsetX, -offsetY);
            }
            return handled;
        }
        transformedEvent = MotionEvent.obtain(event);
    } else {
      	//按在新的view上，事件转换
        transformedEvent = event.split(newPointerIdBits);
    }

    if (child == null) {
      	//自己处理
        handled = super.dispatchTouchEvent(transformedEvent);
    } else {
        final float offsetX = mScrollX - child.mLeft;
        final float offsetY = mScrollY - child.mTop;
        transformedEvent.offsetLocation(offsetX, offsetY);
        if (! child.hasIdentityMatrix()) {
            transformedEvent.transform(child.getInverseMatrix());
        }
				//事件交给子View处理
        handled = child.dispatchTouchEvent(transformedEvent);
    }

    // Done.
    transformedEvent.recycle();
    return handled;
}
```

##### addTouchTarget@ViewGroup

```java
----> addTouchTarget@ViewGroup

private TouchTarget addTouchTarget(@NonNull View child, int pointerIdBits) {
  	//链表前插
  	//mFirstTouchTarget：多指触控时是链表
    final TouchTarget target = TouchTarget.obtain(child, pointerIdBits);
    target.next = mFirstTouchTarget;
    mFirstTouchTarget = target;
    return target;
}
```

##### removePointersFromTouchTargets@ViewGroup

```java
----> removePointersFromTouchTargets@ViewGroup

//清理下touchTarget
//同样的手指再次按下，不一定是同一个view，所以需要清理，根据按下的view重新添加到TouchTarget
private void removePointersFromTouchTargets(int pointerIdBits) {
    TouchTarget predecessor = null;
    TouchTarget target = mFirstTouchTarget;
    while (target != null) {
        final TouchTarget next = target.next;
        if ((target.pointerIdBits & pointerIdBits) != 0) {
            target.pointerIdBits &= ~pointerIdBits;
            if (target.pointerIdBits == 0) {
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
```

#### getAction和getActionMask

##### MotionEvent中常用的事件类型：

- MotionEvent.ACTION_DOWN：第一根手指按下时
- MotionEvent.ACTION_UP：最后一根手指抬起时
- MotionEvent.ACTION_MOVE：按下并移动时（会有多次触发）
- MotionEvent.ACTION_CANCEL：事件取消时（系统用）
- MotionEvent.ACTION_POINTER_DOWN：第二根到第n根手指按下时
- MotionEvent.ACTION_POINTER_UP：倒数n根到倒数第二根手指抬起时

##### 多指触屏时：

- getAction：包含事件类型和手指的index（第几根手指按下抬起）
- getActionMask：会包含事件类型（就是 == getAction & ACTION_MASK，取后八位）

如果只需要处理单指触屏，使用getAction和getActionMask一样，但是想要处理多指触屏，使用getActionMask才能得到准确的事件类型

#### TouchTarget

- 作用：记录每个子view是被哪些pointer（手指）按下的（view + pointerid）
- 结构：单项链表
- pointId分别为（1，10，100，1000）

