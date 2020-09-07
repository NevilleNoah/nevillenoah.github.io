# Android事件分发机制


## Activity事件传递

以“按下”事件为例，首先由Activity捕获点击事件，因此查看`Activity.class`中的`dispatchTouchEvent()`。

```java
// Activity.java
public boolean dispatchTouchEvent(MotionEvent ev) {
  	// 可无视：判断是否是“按下”，如果是则执行onUserInteraction()，这个方法与事件分发机制关联不大。
    if (ev.getAction() == MotionEvent.ACTION_DOWN) { 
        onUserInteraction(); 
    }
  	// 关键：事件分发，跟踪这个方法可以看到事件一层层地传递
    if (getWindow().superDispatchTouchEvent(ev)) {
        return true;
    }
  	// 处理后事：如果事件没有向下传递，那么调用这个方法直接消费事件。
    return onTouchEvent(ev);
}
```



### 事件

在`MotionEvent.class`中所有带`ACTION_`的：

```java
// MotionEvent.java
// Action的掩码
public static final int ACTION_MASK             = 0xff;
// 手指按下
public static final int ACTION_DOWN             = 0;
// 手指抬起
public static final int ACTION_UP               = 1;
// 手指在屏幕上滑动
public static final int ACTION_MOVE             = 2;
// 非人为因素取消
public static final int ACTION_CANCEL           = 3;

// 还有很多其他事件，自己去看
...
```

基本流程：

<img src="image/image-20200907192127083.png" alt="image-20200907192127083" style="zoom:60%;" />





### 用户交互函数

```java
// Activity.java
/**
 * Called whenever a key, touch, or trackball event is dispatched to the
 * activity.  Implement this method if you wish to know that the user has
 * interacted with the device in some way while your activity is running.
 * This callback and {@link #onUserLeaveHint} are intended to help
 * activities manage status bar notifications intelligently; specifically,
 * for helping activities determine the proper time to cancel a notification.
 *
 * <p>All calls to your activity's {@link #onUserLeaveHint} callback will
 * be accompanied by calls to {@link #onUserInteraction}.  This
 * ensures that your activity will be told of relevant user activity such
 * as pulling down the notification pane and touching an item there.
 *
 * <p>Note that this callback will be invoked for the touch down action
 * that begins a touch gesture, but may not be invoked for the touch-moved
 * and touch-up actions that follow.
 *
 * @see #onUserLeaveHint()
 */
public void onUserInteraction() {
}
```

根据注释中给出的信息，`onUserInteraction`会在**按键、触摸、滑动**的情况下调用。

找了`ComponentActivity`、`FragmentActivity`、`AppCompatActivity`，都没有`onUserInteraction`的实现，也就是说这个方法是留工程师使用的。你可以在Activity以及它的任意子类中重写它：

```kotlin
// MainActivity.java
class MainActivity: AppCompatActivity() {
	override fun onUserInteraction() {
		...
	}
}
```



### 事件传递

调用`Window.superDispatchTouchEvent()`进行事件传递。

```java
getWindow().superDispatchTouchEvent(ev)
```



#### PhoneWindow

`getWindow()`获取了`Activity.class`中的`mWindow`变量。

```java
// Activity.java
public Window getWindow() {
	return mWindow;
}
```

`mWindow`在`attach()`方法中被赋值为`PhoneWindow`对象。所以`getWindow()`拿到的是`PhoneWindow`。

```java
// Activity.java
final void attach(...) {
	mWindow = new PhoneWindow(this, window, activityConfigCallback);
}
```

> 源码读到这里，PhoneWindow会报红，直接双击Shift按键，输入PhoneWindow进行搜索。



#### DecorView

`PhoneWindow`中的`superDispatchTouchEvent()`调用了`mDecor`的`superDispatchTouchEvent()`

```java
// PhoneWindow.java
@Override
public boolean superDispatchTouchEvent(MotionEvent event) {
	return mDecor.superDispatchTouchEvent(event);
}
```

`mDecor`是一个`DecorView`对象。

```java
// PhoneWindow.java
private DecorView mDecor;

public PhoneWindow(...) {
	if (preservedWindow != null) {
    ...
  	mDecor = (DecorView) preservedWindow.getDecorView();
    ...
  }
}
```

打开`DecorView`：

* `DecorView.superDispatchTouchEvent()`调用了`ViewGroup.dispatchTouchEvent()`。

* `DecorView`继承`FrameLayout`，包含`ViewGroup mContentRoot`，所以`DecorView`是顶级布局。

```java
// DecorView.java
public class DecorView extends FrameLayout ... {
  // 根内容是ViewGroup
  ViewGroup mContentRoot;
  
  // 继续调用...
  public boolean superDispatchTouchEvent(MotionEvent event) {
     return super.dispatchTouchEvent(event);
  }
}
```

`getWindow()`拿到了`PhoneWindow`对象，`PhoneWindow`中的`DecorView`执行事件`superDispatchTouchEvent`，再进一步传递给`ViewGroup`执行`ViewGroup.dispatchTouchEvent()`。再往下传递就是`ViewGroup`之间的事件传递了，将在后文讲解。



### 处理未消费的事件

```java
return onTouchEvent(ev)
```

调用了`Activity.onTouchEvent()`用来处理没有在传递过程中没有被消费的事件。

```java
// Activity.java
public boolean onTouchEvent(MotionEvent event) {
    if (mWindow.shouldCloseOnTouch(this, event)) {
        finish();
        return true;
    }

    return false;
}
```

调用了`Window.shouldCloseOnTouch()`，方法中判断了方法是否越界、是否按下、是否超出顶级布局，满足条件则Activity消费事件，否则丢弃事件。

```java
// Window.java
public boolean shouldCloseOnTouch(Context context, MotionEvent event) {
      if (mCloseOnTouchOutside && event.getAction() == MotionEvent.ACTION_DOWN
              && isOutOfBounds(context, event) && peekDecorView() != null) {
          return true;
      }
      return false;
}
```



## ViewGroup事件传递

逻辑全部在`ViewGroup.dispatchTouchEvent`中

```java
// ViewGroup.java
@Override
public boolean dispatchTouchEvent(MotionEvent ev) {
	...
}
```



### 检查

在开始正式传递事件前，进行了：一致性检查、焦点检查、隐私政策检查。

```java
// ViewGroup.java
@Override
public boolean dispatchTouchEvent(MotionEvent ev) {
    // 一致性检查：确保手势动作完整
    if (mInputEventConsistencyVerifier != null) {
        mInputEventConsistencyVerifier.onTouchEvent(ev, 1);
    }

    // 焦点检查：确定事件的类型是焦点事件还是普通事件
    if (ev.isTargetAccessibilityFocus() && isAccessibilityFocusedViewOrHost()) {
        ev.setTargetAccessibilityFocus(false);
    }
    
    boolean handled = false;
    // 安全性检查：如果窗口处于隐藏状态则丢弃事件，确保不会触发隐藏窗口
    if (onFilterTouchEventForSecurity(ev)) {
```



#### 一致性检查

```java
// ViewGroup.java
if (mInputEventConsistencyVerifier != null) {
	mInputEventConsistencyVerifier.onTouchEvent(ev, 1);
}
```

`mInputEventConsistencyVerifier`在`View.class`中，在注释中我们看到这个变量是`Debugge`时使用的。

```java
// View.java
/**
 * Consistency verifier for debugging purposes.
 * @hide
 */
protected final InputEventConsistencyVerifier mInputEventConsistencyVerifier =
            InputEventConsistencyVerifier.isInstrumentationEnabled() ?
                    new InputEventConsistencyVerifier(this, 0) : null;
```

在`Debugge`模式下，肯定要启动`Instrumentation`（仪表）用于获取测试信息，因此`isInstrumentationEnabled==true`，则`mInputEventConsistencyVerifier`被赋值为`InputEventConsistencyVerifier`对象，触发`InputEventConsistencyVerifier.onTouchEvent(ev, 1)`来进行**事件的一致性检查**。

在用户模式下，`isInstrumentationEnabled==false`，不做**事件的一致性检查**。在开发者模式下，`isInstrumentationEnabled()==true`如果`ViewGroup`的`isInstrumentationEnabled()`。

什么是**事件的一致性检查**？我们平时用手机，如果你想在屏幕上进行“单击”操作，除了按下还要抬起，才是一个完整“单击”手势，如果只是按下没有抬起，这个手势就不是“单击”手势了。**事件的一致性检查**就是看你有没有与“按下”配对的“抬起”操作。

```java
// InputEventConsistencyVerifier.java
public void onTouchEvent(MotionEvent event, int nestingLevel) {
	switch (action) {
    case MotionEvent . ACTION_DOWN :
    if (mTouchEventStreamPointers != 0) {
        // 如果出现不一致就会报错
        problem("ACTION_DOWN but pointers are already down.  "
                + "Probably missing ACTION_UP from previous gesture.");
    }
    ensureHistorySizeIsZeroForThisAction(event);
    ensurePointerCountIsOneForThisAction(event);
    mTouchEventStreamPointers = 1 < < event . getPointerId (0);
    break;
	}
}
```



#### 焦点检查

```java
// ViewGroup.java
if (ev.isTargetAccessibilityFocus() && isAccessibilityFocusedViewOrHost()) {
	ev.setTargetAccessibilityFocus(false)
}
```

`ev.isTargetAccessibilityFocus()`：事件是否要交给有焦点的View处理。

`isAccessibilityFocusedViewOrHost()`：当前ViewGroup中是否存在有焦点的View。

如果两者都满足，这说明当前ViewGroup的子视图中存在处理**焦点事件**的视图。那么对于当前ViewGroup，这个事件就是**普通事件**。将事件通过`setTargetAccessibilityFocus(false)`转化为普通事件进行处理。

```java
// MotionEvent.java
/** @hide */
public final void setTargetAccessibilityFocus(boolean targetsFocus) {
	final int flags = getFlags();
  nativeSetFlags(mNativePtr, targetsFocus
          ? flags | FLAG_TARGET_ACCESSIBILITY_FOCUS
          : flags & ~FLAG_TARGET_ACCESSIBILITY_FOCUS);
}
```





#### 安全性检查

```java
// ViewGroup.java
    // 安全性检查
    boolean handled = false;
    if (onFilterTouchEventForSecurity(ev)) {
```

使用安全过滤器进行安全性检查

```java
// ViewGroup.java
public boolean onFilterTouchEventForSecurity(MotionEvent event) {
    if ((mViewFlags & FILTER_TOUCHES_WHEN_OBSCURED) != 0
    && (event.getFlags() & MotionEvent.FLAG_WINDOW_IS_OBSCURED) != 0) {
        // 如果当前窗口是隐藏的，那就丢弃这个事件
        return false;
    }
    // 否则消费这个事件
    return true;
}

```



### 拦截器

完成一些简单的参数赋值和状态清空后，我们要确定是否拦截。

```java

// ViewGroup.java
				// 参数赋值
        final int action = ev.getAction();
        final int actionMasked = action & MotionEvent.ACTION_MASK;

				// 重置状态
        if (actionMasked == MotionEvent.ACTION_DOWN) {
            cancelAndClearTouchTargets(ev);
            resetTouchState();
        }

        // 拦截器确定是否拦截
        final boolean intercepted;
        if (actionMasked == MotionEvent.ACTION_DOWN
                || mFirstTouchTarget != null) {
            final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
            if (!disallowIntercept) {
                intercepted = onInterceptTouchEvent(ev);
                ev.setAction(action); 
            } else {
                intercepted = false;
            }
        } else {
            intercepted = true;
        }
```



#### 参数赋值

```java
// ViewGroup.java
        // 参数赋值
        final int action = ev.getAction();
        final int actionMasked = action & MotionEvent.ACTION_MASK;
```

将事件中的`MotionEvent.action`直接赋值给`action`变量

`MotionEvent.action`和`MotionEvent.ACTION_MASK`的并运算结果作为掩码`actionMasked`。



#### 重置状态

```java
// ViewGroup.java
				// 重置状态
        if (actionMasked == MotionEvent.ACTION_DOWN) {
            // 取消和清空所有触摸目标的状态
            cancelAndClearTouchTargets(ev);
            // 重置触摸状态
            resetTouchState();
        }
```

ViewGroup会保留上一次事件的触摸目标的状态，因此开始新的事件的时候，要先将各种状态重置，以便记录新事件的状态。



#### 是否拦截

```java
// ViewGroup.java
				// 拦截器确定是否拦截
        /** 
         * intercepted：拦截标志位
				 * true：已拦截
				 * false：未拦截
				 */
        final boolean intercepted;
        // 默认情况下都允许拦截，如果是"按下"或者"事件曾经经过其他视图【重点，代码段后马上填坑】"，则需要再进一步判断
        // 用到了参数赋值时的actionMaksed变量
				// 子视图可以通过ViewGroup.requestDisallowInterceptTouchEvent()来禁止父视图对事件的拦截
        if (actionMasked == MotionEvent.ACTION_DOWN
                || mFirstTouchTarget != null) {
            // 根据ViewGroup自身的信息确定是否拦截
            final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
            if (!disallowIntercept) {
                // ViewGroup允许拦截，则调用拦截函数进行拦截，并为拦截标志位赋值
                intercepted = onInterceptTouchEvent(ev);
                // 将事件中的action值恢复（前面进行了位操作，要重新赋值还原）
                // 用到了参数赋值时的action变量
                ev.setAction(action); 
            } else {
                // ViewGroup不允许拦截，则不拦截，拦截标志位置false
                intercepted = false;
            }
        } else {
            // 默认情况下都允许拦截，拦截标志位置true
            intercepted = true;
        }
```

`mFirstTouchTarget != null `，为什么意味着“事件曾经经过其他视图”？事件会从父视图向子视图往下传递，如果父视图没有消费掉视图，就会向子视图传递。在这个过程中，会有一个由TouchTarget作为结点组成的单链表，用于存储事件向下传递过程中经过的视图，每经过一个视图就在这个单链表末尾新增一个TouchTarget。例如视图层次从父到子是ABC，则单链表从尾到头是CBA，而`mFirstTouchTarget`就是表尾，`mFirstTouchTarget != null `意味着表不为空，意味着事件曾经经过其他视图。如何得出这个过程？答案在后续的代码中。

拦截函数`onInterceptTouchEvent`

```java
public boolean onInterceptTouchEvent(MotionEvent ev) {
    if (ev.isFromSource(InputDevice.SOURCE_MOUSE) // 事件源来自可点击输入设备（屏幕）
            && ev.getAction() == MotionEvent.ACTION_DOWN // 是“按下”事件
            && ev.isButtonPressed(MotionEvent.BUTTON_PRIMARY) // 是否有屏幕点击或者按键组合
            && isOnScrollbarThumb(ev.getX(), ev.getY()) // 点击的位置是否在视图范围内
    ) {
        // 拦截
        return true;
    }
    // 不拦截
    return false;
}
```



### 分发事件

#### 分发准备

```java
// ViewGroup.java
        // 如果事件已拦截而且曾经经过其他视图，则将事件设定为普通事件
        if (intercepted || mFirstTouchTarget != null) {
            ev.setTargetAccessibilityFocus(false);
        }

        // 取消标志位，检查事件是不是被中途取消
        final boolean canceled = resetCancelNextUpFlag(this)
                || actionMasked == MotionEvent.ACTION_CANCEL;

				// 变量准备
        final boolean split = (mGroupFlags & FLAG_SPLIT_MOTION_EVENTS) != 0;
        TouchTarget newTouchTarget = null; // 新的TouchTarget，用于存储新触摸目标
        boolean alreadyDispatchedToNewTouchTarget = false; // 已分发至新触摸目标标志位
				// 如果事件没被取消且没被拦截，则执行后续代码（如果有符合条件的子视图，就会将事件传递给子视图）
        if (!canceled && !intercepted) {
            View childWithAccessibilityFocus = ev.isTargetAccessibilityFocus()
            ? findChildWithAccessibilityFocus() : null;

            // 对于“按下”事件...
            if (actionMasked == MotionEvent.ACTION_DOWN
                    || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN)
                    || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
                // 事件的索引
                final int actionIndex = ev.getActionIndex(); 
                // 运用事件的索引来获取事件的唯一标识符
                final int idBitsToAssign = split ? 1 << ev.getPointerId(actionIndex) 
                : TouchTarget.ALL_POINTER_IDS;
              
                // 清理掉早前的id，以便后续赋值新的id
                removePointersFromTouchTargets(idBitsToAssign);
```



#### 确定目标

如果不拦截，就找到符合条件的子视图向下传递事件；如果没有符合条件的子视图，那就“另谋出路”，代码段后会解释。

```java
// ViewGroup.java
								// 子视图数量
                final int childrenCount = mChildrenCount;
								// 如果有子视图...
                if (newTouchTarget == null && childrenCount != 0) {
                    // 事件的横坐标
                    final float x = ev.getX(actionIndex);
                    // 事件的纵坐标
                    final float y = ev.getY(actionIndex);
                    // 从前到后遍历子视图，获得的子视图列表
                    final ArrayList<View> preorderedList = buildTouchDispatchChildList();
                    final boolean customOrder = preorderedList == null
                            && isChildrenDrawingOrderEnabled();
                    final View[] children = mChildren;
                    // 从前到后遍历子视图
                    for (int i = childrenCount - 1; i >= 0; i--) {
                        final int childIndex = getAndVerifyPreorderedIndex(
                                childrenCount, i, customOrder);
                        final View child = getAndVerifyPreorderedView(
                                preorderedList, children, childIndex);

                        // 如果子视图无法获取焦点，则跳过本次循环
                        if (childWithAccessibilityFocus != null) {
                            if (childWithAccessibilityFocus != child) {
                                continue;
                            }
                            childWithAccessibilityFocus = null;
                            i = childrenCount - 1;
                        }

                        //如果子视图不可见，或者触摸的坐标不在子视图的范围内，则跳过本次循环
                        if (!child.canReceivePointerEvents()
                                || !isTransformedTouchPointInView(x, y, child, null)) {
                            ev.setTargetAccessibilityFocus(false);
                            continue;
                        }
												
                      	// 经过上述过滤后，找到的第一个符合条件的子视图作为“新的触摸目标”，也就是事件的接收方
                        newTouchTarget = getTouchTarget(child);
                        // 如果有符合条件的视图，那就跳出整个循环
                        if (newTouchTarget != null) {
                            // 把新的id赋值给新的触摸目标
                            newTouchTarget.pointerIdBits |= idBitsToAssign;
                            break;
                        }
												
                      	// 重置取消或抬起标志位
                        resetCancelNextUpFlag(child);
                        // 【重点】将事件传递给第一个符合条件的子视图
                        if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
                            // 子视图接收“按下”事件的时间点
                            mLastTouchDownTime = ev.getDownTime();
                            if (preorderedList != null) {
                                // 从子视图列表中找到子视图的id并记录到mLastTouchDownIndex
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
                            // 【重点】addTouchTarget，将以子视图为内容构建的新结点插入TouchTarget链表的表尾
                            newTouchTarget = addTouchTarget(child, idBitsToAssign);
                            alreadyDispatchedToNewTouchTarget = true;
                            break;
                        }

                        // 将事件设置为普通事件
                        ev.setTargetAccessibilityFocus(false);
                    }
                    // 清理下内存
                    if (preorderedList != null) preorderedList.clear();
                }
								
								// 如果没有找到符合条件的子视图，将TouchTarget单链表表尾的元素mFirstTouchTarget赋值给新触摸目标
                if (newTouchTarget == null && mFirstTouchTarget != null) {
                    newTouchTarget = mFirstTouchTarget;
                    while (newTouchTarget.next != null) {
                        newTouchTarget = newTouchTarget.next;
                    }
                    // 重写分配id
                    newTouchTarget.pointerIdBits |= idBitsToAssign;
                }
            }
        }
```

`addTouchTarget`的代码：

```java
private TouchTarget addTouchTarget(@NonNull View child, int pointerIdBits) {
    final TouchTarget target = TouchTarget.obtain(child, pointerIdBits);
    target.next = mFirstTouchTarget;
    mFirstTouchTarget = target;
    return target;
}
```

前面提到了`mFirstTouchTarget`从而引出了`TouchTarget`单链表的原因，正是因为`addTouchTarget`方法。

`TouchTarget.class`是以`View`为内容的结点。事件不断从父视图传向子视图，每传递一次，子视图收到事件时就会触发一次`addTouchTarget`，子视图就会被构造为新结点，以尾插入的方式插入子节点，并将新插入的结点赋值给`mFirstTouchTarget`。

如果没有找到符合条件的子视图，`mFirstTouchTarget`就是上一次被插入`TouchTarget`单链表的视图，也就是当前视图。

上面这么多代码的目标，就是为了确定`mFirstTouchTarget`作为传递事件的目标，而目标无非是两类：

* 子视图
* 当前视图



### 传递事件

确定`mFirstTouchTarget`之后，开始传递事件，如果是向子视图的传递事件，那已经在上述代码中完成了，下面是当前视图的传递事件。

```java
// ViewGroup.java
        if (mFirstTouchTarget == null) {
            // 【重点】如果没有触摸目标，直接由当前视图进行处理
            handled = dispatchTransformedTouchEvent(ev, canceled, null,
                    TouchTarget.ALL_POINTER_IDS);
        } else {
            // 如果有触摸目标，那么ACTION_MOVE、ACTION_UP等事件开始相继执行。
            TouchTarget predecessor = null;
            TouchTarget target = mFirstTouchTarget;
            while (target != null) {
                final TouchTarget next = target.next;
                if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {
                    handled = true;
                } else {
                    final boolean cancelChild = resetCancelNextUpFlag(target.child)
                            || intercepted;
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

        // 发生抬起或非人为因素取消时，更新触摸目标
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

传递事件过程的主要函数是`dispatchTransformedTouchEvent()`，注意其中的`View.dispatchTouchEvent()`函数的调用。

```java
// ViewGroup.class
private boolean dispatchTransformedTouchEvent(MotionEvent event, boolean cancel,
View child, int desiredPointerIdBits) {
    final boolean handled;

    final int oldAction = event.getAction();
    if (cancel || oldAction == MotionEvent.ACTION_CANCEL) {
        event.setAction(MotionEvent.ACTION_CANCEL);
        if (child == null) {
            // 如果传入的子视图为null，就调用当前视图的dispatchTouchEvent()，相当于分发给当前视图。
            handled = super.dispatchTouchEvent(event);
        } else {
            // 如果传入的子视图不为null，则执行子视图的dispatchTouchEvent()，相当于分发给子视图。
            handled = child.dispatchTouchEvent(event);
        }
        event.setAction(oldAction);
        return handled;
    }

    ...
    
    return handled;
}
```

注意，这里调用的不是`ViewGroup.dispatchTouchEvent()`而是`View.dispatchTouchEvent()`。从个方法开始进入View的事件传递。



## View事件传递

```java
// View.class
public boolean dispatchTouchEvent(MotionEvent event) {
   // 焦点事件的处事方式
    if (event.isTargetAccessibilityFocus()) {
        if (!isAccessibilityFocusedViewOrHost()) {
            return false;
        }
        // 处理完后转为普通事件
        event.setTargetAccessibilityFocus(false);
    }

    boolean result = false;
		// 一致性检查
    if (mInputEventConsistencyVerifier != null) {
        mInputEventConsistencyVerifier.onTouchEvent(event, 0);
    }

    final int actionMasked = event.getActionMasked();
    if (actionMasked == MotionEvent.ACTION_DOWN) {
        // 如果存在滚动操作，立刻停止
        stopNestedScroll();
    }

    // 安全性检查
    if (onFilterTouchEventForSecurity(event)) {
        if ((mViewFlags & ENABLED_MASK) == ENABLED && handleScrollBarDragging(event)) {
            result = true;
        }
        ListenerInfo li = mListenerInfo;
        // 先执行OnTouchListener.onTouch()消费事件
        if (li != null && li.mOnTouchListener != null
                && (mViewFlags & ENABLED_MASK) == ENABLED
        && li.mOnTouchListener.onTouch(this, event)) {
            result = true;
        }
        // 如果OnTouchListener.onTouch()消费事件没有消费事件，再执行onTouchEvent()消费事件
        if (!result && onTouchEvent(event)) {
            result = true;
        }
    }

    if (!result && mInputEventConsistencyVerifier != null) {
        mInputEventConsistencyVerifier.onUnhandledEvent(event, 0);
    }

    if (actionMasked == MotionEvent.ACTION_UP ||
            actionMasked == MotionEvent.ACTION_CANCEL ||
            (actionMasked == MotionEvent.ACTION_DOWN && !result)) {
        stopNestedScroll();
    }

    return result;
}
```

`View.onTouchEvent()`代码，大致逻辑：如果视图可以被点击或者长按并且在特定手势下满足一些特定条件，则返回true，表示已经被消费；否则返回false，表示未被消费。

```java
public boolean onTouchEvent(MotionEvent event) {
    final float x = event.getX();
    final float y = event.getY();
    final int viewFlags = mViewFlags;
    final int action = event.getAction();

    final boolean clickable = ((viewFlags & CLICKABLE) == CLICKABLE
    || (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE)
    || (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE;

    
    if ((viewFlags & ENABLED_MASK) == DISABLED) {
        if (action == MotionEvent.ACTION_UP && (mPrivateFlags & PFLAG_PRESSED) != 0) {
        setPressed(false);
    }
        mPrivateFlags3 &= ~PFLAG3_FINGER_DOWN;
        // A disabled view that is clickable still consumes the touch
        // events, it just doesn't respond to them.
        return clickable;
    }
    if (mTouchDelegate != null) {
        if (mTouchDelegate.onTouchEvent(event)) {
            return true;
        }
    }

    if (clickable || (viewFlags & TOOLTIP) == TOOLTIP) {
        switch (action) {
            case MotionEvent.ACTION_UP:
            mPrivateFlags3 &= ~PFLAG3_FINGER_DOWN;
            if ((viewFlags & TOOLTIP) == TOOLTIP) {
            handleTooltipUp();
        }
            if (!clickable) {
                removeTapCallback();
                removeLongPressCallback();
                mInContextButtonPress = false;
                mHasPerformedLongPress = false;
                mIgnoreNextUpEvent = false;
                break;
            }
            boolean prepressed = (mPrivateFlags & PFLAG_PREPRESSED) != 0;
            if ((mPrivateFlags & PFLAG_PRESSED) != 0 || prepressed) {
            // take focus if we don't have it already and we should in
            // touch mode.
            boolean focusTaken = false;
            if (isFocusable() && isFocusableInTouchMode() && !isFocused()) {
                focusTaken = requestFocus();
            }

            if (prepressed) {
                // The button is being released before we actually
                // showed it as pressed.  Make it show the pressed
                // state now (before scheduling the click) to ensure
                // the user sees it.
                setPressed(true, x, y);
            }

            if (!mHasPerformedLongPress && !mIgnoreNextUpEvent) {
                // This is a tap, so remove the longpress check
                removeLongPressCallback();

                // Only perform take click actions if we were in the pressed state
                if (!focusTaken) {
                    // Use a Runnable and post this rather than calling
                    // performClick directly. This lets other visual state
                    // of the view update before click actions start.
                    if (mPerformClick == null) {
                        mPerformClick = new PerformClick();
                    }
                    if (!post(mPerformClick)) {
                        performClickInternal();
                    }
                }
            }

            if (mUnsetPressedState == null) {
                mUnsetPressedState = new UnsetPressedState();
            }

            if (prepressed) {
                postDelayed(mUnsetPressedState,
                        ViewConfiguration.getPressedStateDuration());
            } else if (!post(mUnsetPressedState)) {
                // If the post failed, unpress right now
                mUnsetPressedState.run();
            }

            removeTapCallback();
        }
            mIgnoreNextUpEvent = false;
            break;

            case MotionEvent.ACTION_DOWN:
            if (event.getSource() == InputDevice.SOURCE_TOUCHSCREEN) {
                mPrivateFlags3 |= PFLAG3_FINGER_DOWN;
            }
            mHasPerformedLongPress = false;

            if (!clickable) {
                checkForLongClick(
                        ViewConfiguration.getLongPressTimeout(),
                        x,
                        y,
                        TOUCH_GESTURE_CLASSIFIED__CLASSIFICATION__LONG_PRESS);
                break;
            }

            if (performButtonActionOnTouchDown(event)) {
                break;
            }

            // Walk up the hierarchy to determine if we're inside a scrolling container.
            boolean isInScrollingContainer = isInScrollingContainer();

            // For views inside a scrolling container, delay the pressed feedback for
            // a short period in case this is a scroll.
            if (isInScrollingContainer) {
                mPrivateFlags |= PFLAG_PREPRESSED;
                if (mPendingCheckForTap == null) {
                    mPendingCheckForTap = new CheckForTap();
                }
                mPendingCheckForTap.x = event.getX();
                mPendingCheckForTap.y = event.getY();
                postDelayed(mPendingCheckForTap, ViewConfiguration.getTapTimeout());
            } else {
                // Not inside a scrolling container, so show the feedback right away
                setPressed(true, x, y);
                checkForLongClick(
                        ViewConfiguration.getLongPressTimeout(),
                        x,
                        y,
                        TOUCH_GESTURE_CLASSIFIED__CLASSIFICATION__LONG_PRESS);
            }
            break;

            case MotionEvent.ACTION_CANCEL:
            if (clickable) {
                setPressed(false);
            }
            removeTapCallback();
            removeLongPressCallback();
            mInContextButtonPress = false;
            mHasPerformedLongPress = false;
            mIgnoreNextUpEvent = false;
            mPrivateFlags3 &= ~PFLAG3_FINGER_DOWN;
            break;

            case MotionEvent.ACTION_MOVE:
            if (clickable) {
                drawableHotspotChanged(x, y);
            }

            final int motionClassification = event.getClassification();
            final boolean ambiguousGesture =
                    motionClassification == MotionEvent.CLASSIFICATION_AMBIGUOUS_GESTURE;
            int touchSlop = mTouchSlop;
            if (ambiguousGesture && hasPendingLongPressCallback()) {
                final float ambiguousMultiplier =
                        ViewConfiguration.getAmbiguousGestureMultiplier();
                if (!pointInView(x, y, touchSlop)) {
                    // The default action here is to cancel long press. But instead, we
                    // just extend the timeout here, in case the classification
                    // stays ambiguous.
                    removeLongPressCallback();
                    long delay = (long) (ViewConfiguration.getLongPressTimeout()
                            * ambiguousMultiplier);
                    // Subtract the time already spent
                    delay -= event.getEventTime() - event.getDownTime();
                    checkForLongClick(
                            delay,
                            x,
                            y,
                            TOUCH_GESTURE_CLASSIFIED__CLASSIFICATION__LONG_PRESS);
                }
                touchSlop *= ambiguousMultiplier;
            }

            // Be lenient about moving outside of buttons
            if (!pointInView(x, y, touchSlop)) {
                // Outside button
                // Remove any future long press/tap checks
                removeTapCallback();
                removeLongPressCallback();
                if ((mPrivateFlags & PFLAG_PRESSED) != 0) {
                    setPressed(false);
                }
                mPrivateFlags3 &= ~PFLAG3_FINGER_DOWN;
            }

            final boolean deepPress =
                    motionClassification == MotionEvent.CLASSIFICATION_DEEP_PRESS;
            if (deepPress && hasPendingLongPressCallback()) {
                // process the long click action immediately
                removeLongPressCallback();
                checkForLongClick(
                        0 /* send immediately */,
                        x,
                        y,
                        TOUCH_GESTURE_CLASSIFIED__CLASSIFICATION__DEEP_PRESS);
            }

            break;
        }

        return true;
    }

    return false;
}
```



## 总结

事件分发顺序：Activity=>ViewGroup=>View

Activity事件传递机制：Activity通过`PhoneWindow`将事件传递给顶级布局`DecorView`，顶级布局将事件传给它内含的`ViewGroup`，调用`ViewGroup`的`ViewGroup.dispatchTouchEvent()`来分发事件。

ViewGroup事件传递机制：`ViewGroup.dispatchTouchEvent()`中先通过`ViewGroup.onInterceptTouchEvent()`判断是否拦截，如果拦截则将事件传递给子视图，不拦截则将事件传递给当前视图，从而确定触摸目标`mFirstTouchTarget`是子视图还是当前视图。再调用`ViewGroup.dispatchTransformedTouchEvent()`将事件分发给`mFirstTouchTarget`，当前视图和子视图都通过调用`View.dispatchTouchEvent()`来消费事件。

View事件传递机制：`View.dispatchTouchEvent()`中通过调用`OnTouchListener.onTouch()`消费事件，若成功消费则返回true，否则调用`View.onTouchEvent()`事件并返回false，并且层层返回false，最后由`Activity.onTouchEvent()`来消费事件。

<img src="image/image-20200907171148737.png" alt="image-20200907171148737" style="zoom:100%;" />




