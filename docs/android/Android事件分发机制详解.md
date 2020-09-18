# Android事件分发机制详解

# 1. 什么是事件

要了解事件分发，那我们先说说什么是事件，其实这里的事件指的就是点击事件，当用户触摸屏幕的时候，将会产生点击事件（`Touch`事件）

`Touch`事件的相关细节（发生触摸的位置、时间等）被封装成`MotionEvent`对象

**MotionEvent事件类型**

| 事件类型                  | 具体动作                   |
| ------------------------- | -------------------------- |
| MotionEvent.ACTION_DOWN   | 按下View（所有事件的开始） |
| MotionEvent.ACTION_UP     | 抬起View（与DOWN对应）     |
| MotionEvent.ACTION_MOVE   | 滑动View                   |
| MotionEvent.ACTION_CANCEL | 结束事件（非人为原因）     |

**事件序列**：其实就是从手指触摸屏幕到离开屏幕所发生的一系列事件

# 2. 什么是事件分发

我们要讲的事件分发其实就是将点击事件传递到某个具体的`View`，这个传递的过程就叫做事件分发

# 3. 事件在哪些对象间进行传递、顺序是什么

`Activity`的`UI`界面由`Activity`、`ViewGroup`、`View`及其派生类组成



![img](https://user-gold-cdn.xitu.io/2019/7/29/16c3b74f8063d008?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



事件分发在这三个对象之间进行传递。

当点击事件发生后，事件先传到`Activity`，再传到`ViewGroup`，最终传到`View`

# 4. 事件分发有啥用？

默认情况下事件分发会按照由`Activity`到`ViewGroup`再到`View`的顺序进行分发，当我们不想`View`进行处理，让`ViewGroup`处理，那就可以进行拦截，这些知识可以用于解决滑动冲突。

例如：外部滑动和内部滑动方向不一致，当`ScrollView`嵌套`Fragment`，且`Fragemnt`内部有个竖向的`ListView`，当用户左右滑动时，要让外部的`View`拦截单击事件，当用户上下滑动时，要让内部的`View`拦截点击事件。怎么拦截，在哪里拦截，就用到了我们这篇文章所讲的内容了。



![img](https://user-gold-cdn.xitu.io/2019/7/29/16c3b7591e422816?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



# 5. 事件分发涉及到的函数及相应的作用

| 方法                  | 作用                             |
| --------------------- | -------------------------------- |
| dispatchTouchEvent    | 进行事件分发                     |
| onInterceptTouchEvent | 事件拦截                         |
| onTouchEvent          | 事件消耗（就是交给当前View处理） |

- **dispatchTouchEvent：** 用来进行事件分发，若事件能够传递到当前`View`，则此方法一定会被调用。
- **onInterceptTouchEvent：** 在`dispatchTouchEvent`方法内被调用，用来判断是否拦截某个事件。若当前`View`拦截了某个事件，则该方法不会再被调用，**返回结果表示是否拦截当前事件**，该方法只在`ViewGroup`中存在。
- **onTouchEvent：** 用来处理点击事件，返回结果表示是否消耗当前事件，若不消耗，则在同一事件序列中，当前`View`无法再次接收到事件。

这三个方法可用以下伪代码表示

```
public boolean dispatchTouchEvent(MotionEvent ev){
    boolean consume = false;
    if(onInterceptTouchEvent(ev)){
        consume = onTouchEvent(ev);
    }else{
        consume = child.dispatchTouchEvent(ev);
    }
    return consume;
}
复制代码
```

对应的根`ViewGroup`，当一个点击事件产生时，`Activity`会传递给它，这时它的`dispatchTouchEvent`就会被调用，若该`ViewGroup`的`onInterceptTouchEvent`返回`true`，代表拦截该事件，但是否消耗该事件，还要看它的`onTouchEvent`的返回值，如果不拦截，则代表将事件分发下去给子`View`，接着子`View`的`dispatchTouchEvent`方法会被调用，如此反复直到事件被最终处理。

# 6. Activity的事件分发

当一个点击事件发生时，事件最先传到`Activity`的`dispatchTouchEvent()`进行事件分发。这里主要要弄明白`Activity`是怎么将事件分发到`ViewGroup`中的

## 6.1 Demo演示

我们先看一个案例

**（1）** 自定义一个`MyViewGroup`，继承自`ViewGroup`，重写`dispatchTouchEvent()`方法

```
public class MyViewGroup extends ViewGroup{
    
    public boolean dispatchTouchEvent(MotionEvent ev) {
		Log.i(TAG, "dispatchTouchEvent: ");        
		//这里我们暂时先返回false		
		return false;
    }
}
复制代码
```

**（2）** 在`Activity`的布局中，使用该布局作为最外层布局

```
<com.ld.eventdispatchdemo.activitydispatch.MyViewGroup xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:id="@+id/myViewGroup"
    android:orientation="vertical"
    tools:context=".activitydispatch.ActivityDispatchActivity">

</com.ld.eventdispatchdemo.activitydispatch.MyViewGroup>
复制代码
```

**（3）** 重写该`Activity`的`dispatchTouchEvent()`和`onTouchEvent()`方法，打印log日志

```
public class Activity extends AppCompatActivity{
    ...
	private static final String TAG = "Activit_activitydispatch";
    
    @Override
    public boolean dispatchTouchEvent(MotionEvent ev) {

        if(ev.getAction()==MotionEvent.ACTION_DOWN){
            Log.i(TAG, "dispatchTouchEvent: ");
        }
     	//这里是仿照源码的格式写的
        if(getWindow().superDispatchTouchEvent(ev)){
            Log.i(TAG, "dispatchTouchEvent: 这里被调用");
            return true;
        }        
        return onTouchEvent(ev);
    }       
    
    @Override
    public boolean onTouchEvent(MotionEvent event) {
        Log.i(TAG, "onTouchEvent: ");
        return super.onTouchEvent(event);
    }
}
复制代码
```

当`MyViewGroup`的`dispatchTouchEvent`返回`false`时，打印的log日志为：

```
Activit_activitydispatch: dispatchTouchEvent: 
MyViewGroup_activitydispatch: dispatchTouchEvent: 
Activit_activitydispatch: onTouchEvent: 
Activit_activitydispatch: onTouchEvent: 
Activit_activitydispatch: onTouchEvent: 
Activit_activitydispatch: onTouchEvent: 
Activit_activitydispatch: onTouchEvent: 
复制代码
```

当`MyViewGroup`的`dispatchTouchEvent`返回`true`时，打印的log日志为：

```
Activit_activitydispatch: dispatchTouchEvent: 
MyViewGroup_activitydispatch: dispatchTouchEvent: 
Activit_activitydispatch: dispatchTouchEvent: 这里被调用
MyViewGroup_activitydispatch: dispatchTouchEvent: 
Activit_activitydispatch: dispatchTouchEvent: 这里被调用
MyViewGroup_activitydispatch: dispatchTouchEvent: 
Activit_activitydispatch: dispatchTouchEvent: 这里被调用
MyViewGroup_activitydispatch: dispatchTouchEvent: 
Activit_activitydispatch: dispatchTouchEvent: 这里被调用
复制代码
```

仔细观察可以看到，**当MyViewGroup的dispatchTouchEvent返回false时，Activity的onTouchEvent会被调用，返回true时，不会被调用，这是什么原因呢？**

你可能会有疑问，我的`Activity`的`dispatchTouchEvent()`方法内为何要这样写呢？别急，看完下面的源码你就知道了。

## 6.2 源码解析

**目的：**

1、研究`MyViewGroup`的`dispatchTouchEvent`返回`false`时，`Activity`的`onTouchEvent`会被调用，返回`true`时，不会被调用的原因

`Activity`中的`dispatchTouchEvent()`源码如下：

```
public boolean dispatchTouchEvent(MotionEvent ev) {
    if (ev.getAction() == MotionEvent.ACTION_DOWN) {
        //该方法为空方法，不用管它
        onUserInteraction();
    }
    if (getWindow().superDispatchTouchEvent(ev)) {
        return true;
    }
    return onTouchEvent(ev);
}
复制代码
```

有没有很熟悉，为了方便观察打日志所以上面我们重写了`Activity`的`dispatchTouchEvent`方法，但内容和源码基本一致。

可以从源码看到，当`getWindow().superDispatchTouchEvent(ev)==true`时，那么此时`return ture`，自然就不会调用底下的`onTouchEvent()`方法，即`Activity`的`onTouchEvent()`。

`getWindow()`返回`Window`对象，`Window`是抽象类，而`PhoneWindow`是`Window`的唯一实现类，所以`getWindow().superDispatchTouchEvent(ev)`其实就是调用的`PhoneWindow`内的`superDispatchTouchEvent(ev)`方法。

看看`PhoneWindow`类源码：

```
@Override
public boolean superDispatchTouchEvent(MotionEvent event) {
    return mDecor.superDispatchTouchEvent(event);
}
复制代码
```

`PhoneWindow`将事件直接传递给了`DecorView`，接下来看看`DecorView`是什么

```
public class DecorView extends FrameLayout implements RootViewSurfaceTaker, WindowCallbacks {
                     
}
复制代码
```

`mDecor`是`getWindow().getDecorView()`返回的`View`，通过`setContentView`设置的`View`是该`View`的子`View`。

`DecorView`继承自`FrameLayout(ViewGroup)`，所以`mDecor.superDispatchTouchEvent(event)`其实调用的就是`ViewGroup`的`dispatchTouchEvent()`方法，所以到这里你就懂了吧，

```
if (getWindow().superDispatchTouchEvent(ev)) {
    return true;
}
复制代码
```

其实就相当于下面这个

```
if(viewgroup.DispatchTouchEvent(ev)){
    return true;
}
复制代码
```

所以说当我们的`MyLayout`的`DispatchTouchEvent()`返回`true`时，`Activity`的`onTouchEvent`就不会被调用。

## 6.3 事件时怎么从Activity分发到ViewGroup中的

从上面`Activity`的`dispatchTouchEvent`源码可知道，默认状态下，它内部一定会调用该方法，而`if()`条件中的内容其实就是调用`ViewGroup`的`dispatchTouchEvent()`方法，也就是在这里完成了`Activity`到`ViewGroup`的事件分发。

```
if (getWindow().superDispatchTouchEvent(ev)) {
    return true;
}
复制代码
```

## 6.4 小结：Activity分发的流程图



![img](https://user-gold-cdn.xitu.io/2019/7/29/16c3b797649e4913?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



# 7. ViewGroup的事件分发

上面讲了`Activity`在`dispatchTouchEvent`内将事件传递到了`ViewGroup`的`dispatchTouchEvent()`方法中，那么`ViewGroup`又是如何将事件进一步向下分发的呢？

## 7.1 Demo演示

**（1）** 自定义`MyLayout`，继承自`LinearLayout`，重写`onInterceptTouchEvent()`方法，并返回`true`，重写`dispatchTouchEvent()`、`onTouchEvent()`方法，打印log日志

```
public class MyLayout extends LinearLayout {
    
    private static final String TAG = "MyLayout_ViewGroupDispatch";
    
    @Override
    public boolean dispatchTouchEvent(MotionEvent ev) {
        Log.i(TAG, "dispatchTouchEvent: ");
        return super.dispatchTouchEvent(ev);
    }
    
    @Override
    public boolean onInterceptTouchEvent(MotionEvent ev) {
        Log.i(TAG, "onInterceptTouchEvent: ");
        //此处暂时返回true观察现象
        return true;
    }
    
    @Override
    public boolean onTouchEvent(MotionEvent event) {
        Log.i(TAG, "onTouchEvent: ");
        return super.onTouchEvent(event);
    }   
}
复制代码
```

**（2）** 在`Activity`的布局中，使用该布局作为最外层布局，并在该布局内添加一个按钮

```
<com.ld.eventdispatchdemo.viewgroupdispatch.MyLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    android:id="@+id/myLayout"
    tools:context=".viewgroupdispatch.ViewGroupActivity">
    <Button
        android:id="@+id/btn1"
        android:text="Button1"
        android:layout_width="match_parent"
        android:layout_height="wrap_content" />  
</com.ld.eventdispatchdemo.viewgroupdispatch.MyLayout>
复制代码
```

**（3）** 在`Activity`内为按钮添加点击事件

```
public class ViewGroupActivity extends AppCompatActivity {
    
    private Button btn1;
    private static final String TAG = "Activity_ViewGroupDispatch";
	@Override
    protected void onCreate(Bundle savedInstanceState) {
        
        btn1 = findViewById(R.id.btn1);
        btn1.setOnClickListener(new View.OnClickListener() {
            @SuppressLint("LongLogTag")
            @Override
            public void onClick(View v) {
                Log.i(TAG, "onClick: 点击了按钮1");
            }
        });
    }           
}
复制代码
```

当`MyLayout`的`onInterceptTouchEvent()`返回`true`时，分别点击空白处、点击按钮，log日志如下：

```
//点击空白处
MyLayout_ViewGroupDispatch: dispatchTouchEvent: 
MyLayout_ViewGroupDispatch: onInterceptTouchEvent: 
MyLayout_ViewGroupDispatch: onTouchEvent: 
//点击按钮
MyLayout_ViewGroupDispatch: dispatchTouchEvent: 
MyLayout_ViewGroupDispatch: onInterceptTouchEvent: 
MyLayout_ViewGroupDispatch: onTouchEvent: 
复制代码
```

当`MyLayout`的`onInterceptTouchEvent()`返回`false`时，分别点击空白处、点击按钮，log日志如下：

```
//点击空白处
MyLayout_ViewGroupDispatch: dispatchTouchEvent: 
MyLayout_ViewGroupDispatch: onInterceptTouchEvent: 
MyLayout_ViewGroupDispatch: onTouchEvent: 
//点击按钮
MyLayout_ViewGroupDispatch: onInterceptTouchEvent: 
MyLayout_ViewGroupDispatch: dispatchTouchEvent: 
MyLayout_ViewGroupDispatch: onInterceptTouchEvent: 
MyLayout_ViewGroupDispatch: dispatchTouchEvent: 
MyLayout_ViewGroupDispatch: onInterceptTouchEvent: 
MyLayout_ViewGroupDispatch: dispatchTouchEvent: 
MyLayout_ViewGroupDispatch: onInterceptTouchEvent: 
MyLayout_ViewGroupDispatch: dispatchTouchEvent: 
MyLayout_ViewGroupDispatch: onInterceptTouchEvent: 
Activity_ViewGroupDispatch: onClick: 点击了按钮1
复制代码
```

可以看到当`ViewGroup(MyLayout)`的`onInterceptTouchEvent()`返回`true`时，并没有触发按钮的点击事件，并且自身的`onTouchEvent()`方法被调用，当返回`false`时，按钮的点击事件触发，但自身的`onTouchEvent()`方法未被调用。

而且在默认状态下，`onInterceptTouchEvent()`一定会被调用。

以上现象是什么原因呢？接下来我们看看`ViewGroup的dispatchTouchEvent()`方法的源码

## 7.2 源码解析

**目的：**

1、研究当`ViewGroup(MyLayout)`的`onInterceptTouchEvent()`返回`true`时，并没有触发按钮的点击事件，并且自身的`onTouchEvent()`方法被调用，当返回`false`时，按钮的点击事件触发，但自身的`onTouchEvent()`方法未被调用的原因

2、分发事件是怎么从`ViewGroup`分发到`View`中的

`ViewGroup`的`dispatchTouchEvent()`源码：

```
@Override
public boolean dispatchTouchEvent(MotionEvent ev) {
	...
        
    boolean handled = false;
    if (onFilterTouchEventForSecurity(ev)) {
  		......
         //一大堆代码   
    }
    
    if (!handled && mInputEventConsistencyVerifier != null) {
        mInputEventConsistencyVerifier.onUnhandledEvent(ev, 1);
    }
    return handled;
}
复制代码
```

我们想要了解事件是如何分发，其实是主要看`ViewGroup`的`dispatchTouchEvent()`方法什么时候返回`true`，什么时候返回`false`。看源码可以知道，`ViewGroup`的`dispatchTouchEvent()`方法返回的是`handle`的值，所以我们只需要观察该方法内改变`handle`值的语句。

首先初始化了`handle`的值，默认为`false`

然后你可以看到`dispatchTouchEvent()`的大部分内容都在`if (onFilterTouchEventForSecurity(ev)) {}`这个条件判断内，也就是说如果`onFilterTouchEventForSecurity(ev)`方法返回`true`的话，那么就进入该`if`判断内。若返回`false`，则`dispatchTouchEvent()`返回初始值为`false`的`handled`，表示不分发事件。

查看下`onFilterTouchEventForSecurity(ev)`方法

```
public boolean onFilterTouchEventForSecurity(MotionEvent event) {
    //noinspection RedundantIfStatement
    if ((mViewFlags & FILTER_TOUCHES_WHEN_OBSCURED) != 0
        && (event.getFlags() & MotionEvent.FLAG_WINDOW_IS_OBSCURED) != 0) {
        // Window is obscured, drop this touch.
        return false;
    }
    return true;
}
复制代码
```

- `FILTER_TOUCHES_WHEN_OBSCURED`是`android:filterTouchesWhenObscured`属性所对应的。`android:filterTouchesWhenObscured`是`true`的话，则表示其他视图在该视图之上，导致该视图被隐藏时，该视图就不再响应触摸事件。
- `MotionEvent.FLAG_WINDOW_IS_OBSCURED`为`true`的话，则表示该视图的窗口是被隐藏的

而我们并没有在`XML`中为控件设置`android:filterTouchesWhenObscured`属性，所以它==0，没有进入`if()`方法，所以`onFilterTouchEventForSecurity()`方法返回`true`，那么`if (onFilterTouchEventForSecurity(ev))`判断必定会进入如下的判断中。

接下来我们看看`if(onFilterTouchEventForSecurity(ev))`判断下的内容

```
if (onFilterTouchEventForSecurity(ev)) {
    final int action = ev.getAction();
    final int actionMasked = action & MotionEvent.ACTION_MASK;
    
    if (actionMasked == MotionEvent.ACTION_DOWN) {        
        cancelAndClearTouchTargets(ev);
        
        resetTouchState();       
    }   
    
 	......   
}
复制代码
private void resetTouchState() {
    
    clearTouchTargets();
    
    resetCancelNextUpFlag(this);
    mGroupFlags &= ~FLAG_DISALLOW_INTERCEPT;
    mNestedScrollAxes = SCROLL_AXIS_NONE;
}
复制代码
private void clearTouchTargets() {
    TouchTarget target = mFirstTouchTarget;
    if (target != null) {
        do {
            TouchTarget next = target.next;
            target.recycle();
            target = next;
        } while (target != null);
        mFirstTouchTarget = null;
    }
}
复制代码
```

可以看到，若为`ACTION_DOWN`事件，就会触发 `cancelAndClearTouchTargets(ev)`和`resetTouchState()`方法，在`resetTouchState()`方法中，有一个`clearTouchTargets()`方法，而在 `clearTouchTargets()`方法内会将`mFirstTouchTarget`设置为`null`。我们暂时先记住这个`mFristTouchTarget`已经置为`null`了。

我们再看`if (onFilterTouchEventForSecurity(ev)){}`该判断内的其他代码

```
if (onFilterTouchEventForSecurity(ev)) {
    ......
    //记录是否拦截    
    final boolean intercepted;
    
    if (actionMasked == MotionEvent.ACTION_DOWN
        || mFirstTouchTarget != null) {
        
        //判断是否设置了FLAG_DISALLOW_INTERCEPT这个标记位,默认为false
        //disallowIntercept代表禁止拦截判断
        final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
        
        if (!disallowIntercept) {
            //关键看这里
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
复制代码
```

前面我们知道了`mFirstTouchTarget`为`null`，所以说只要是`ACTION_DOWN`事件，就会进入到`if (actionMasked == MotionEvent.ACTION_DOWN|| mFirstTouchTarget != null) {}`该方法块内，因为我们没有设置`FLAG_DISALLOW_INTERCEPT`属性，所以它为默认为`false`，所以进入到了`if (!disallowIntercept) {}`方法块内，调用了`onInterceptTouchEvent()`方法。这里就解释了为何默认情况下`dispatchTouchEvent()`后会调用`onInterceptTouchEvent()`方法。

接下来我们看下`if (onFilterTouchEventForSecurity(ev)) {}`方法块其余部分源码

```
if (onFilterTouchEventForSecurity(ev)) {
    ......
        if (!canceled && !intercepted) {
        final View[] children = mChildren;
            
        for (int i = childrenCount - 1; i >= 0; i--) {
            final int childIndex = getAndVerifyPreorderedIndex(
                childrenCount, i, customOrder);
            final View child = getAndVerifyPreorderedView(
                preorderedList, children, childIndex);
            
       		//判断子元素是否能够接受点击事件
            if (!canViewReceivePointerEvents(child)
                || !isTransformedTouchPointInView(x, y, child, null)) {
                ev.setTargetAccessibilityFocus(false);
                continue;
            }
            
            newTouchTarget = getTouchTarget(child);
            if (newTouchTarget != null) {                      
                newTouchTarget.pointerIdBits |= idBitsToAssign;
                break;
            }
            
            resetCancelNextUpFlag(child);
            //调用子元素的dispatchTouchEvent方法
            if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
                // Child wants to receive touch within its bounds.
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
                newTouchTarget = addTouchTarget(child, idBitsToAssign);
                alreadyDispatchedToNewTouchTarget = true;
                break;
            }                           
    }            
}
复制代码
```

在上一个方法块中，`intercepted=onInterceptTouchEvent()`的返回值，当不拦截的时候，`intercepted==false`，进入到`if (!canceled && !intercepted) {}`方法块内。可以看到我们通过`for`循环遍历了所有的子元素，然后判断子元素是否能够接收到点击事件。判断子元素是否能够接收点击事件由两点决定：1、`canViewReceivePointerEvents(child)`判断点击事件坐标是否落在子元素的区域内，2、`isTransformedTouchPointInView(x, y, child, null)`判断子元素是否在播放动画。

所以说当`onInterceptTouchEvent()`返回`false`时，触发了点击事件，返回`true`时没有触发。

若满足这两个条件，则事件传递给它处理。有一个为`ture`则不会进入到该`if (!canceled && !intercepted) {}`方法块内，而是执行下面的代码。

`if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {}`这个方法块，`dispatchTransformedTouchEvent()`其实是调用子元素的`dispatchTouchEvent()`方法，`dispatchTransformedTouchEvent()`源码如下：

```
private boolean dispatchTransformedTouchEvent(MotionEvent event, boolean cancel,View child, int desiredPointerIdBits) {
    
    ......
    if (child == null) {
        //child为null,调用父类的dispatchTouchEvent方法，ViewGroup父类为View，所以是调用View的dispatchTouchEvent方法。
        handled = super.dispatchTouchEvent(transformedEvent);
    } else {
        final float offsetX = mScrollX - child.mLeft;
        final float offsetY = mScrollY - child.mTop;
        transformedEvent.offsetLocation(offsetX, offsetY);
        if (! child.hasIdentityMatrix()) {
            transformedEvent.transform(child.getInverseMatrix());
        }
        
       	//child不为null      
        handled = child.dispatchTouchEvent(transformedEvent);
    }
    ......

}
复制代码
```

可以看到当`child`不为`null`时，调用`child.dispatchTouchEvent(transformedEvent)`，完成了从`ViewGroup`到`View`的事件分发。

## 7.3 事件怎么从ViewGroup分发到View中的

在上面的源码中，`ViewGroup`的`dispatchTouchEvent`方法内，当`onInterceptTouchEvent`返回`false`时，会调用`dispatchTransformedTouchEvent()`方法，而该方法内会调用`View`的`dispatchTouchEvent`，在这里实现了事件从`ViewGroup`到`View`的事件分发。

## 7.4 小结：ViewGroup分发的流程图



![img](https://user-gold-cdn.xitu.io/2019/7/29/16c3b7c84fc2ff81?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



# 8. View的事件分发

## 8.1 Demo演示

**（1）** 自定义一个`MyButton`，继承自`Button`，重写`dispatchTouchEvent()`和`onTouchEvent()`方法并打印日志

```
public class MyButton extends AppCompatButton {
    
    @Override
    public boolean dispatchTouchEvent(MotionEvent event) {        
        Log.i(TAG, "dispatchTouchEvent: ");        
        return super.dispatchTouchEvent(event);
    }

    @Override
    public boolean onTouchEvent(MotionEvent event) {

        switch (event.getAction()) {
            case MotionEvent.ACTION_DOWN:
                Log.i(TAG, "onTouchEvent: ACTION_DOWN");
                break;
            default:
                break;
        }
        return super.onTouchEvent(event);
    }       
}
复制代码
```

**（2）** 在`Activity`布局中，放入该控件

```
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".viewdiapatch.ViewDispatchActivity">

    <com.ld.eventdispatchdemo.viewdiapatch.MyButton   
        android:id="@+id/btn_click"
        android:text="view的点击事件分发"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:clickable="true"/>
</LinearLayout>
复制代码
```

**（3）** 在`Activity`内为按钮添加点击事件和`touch`事件

```
public class ViewDispatchActivity extends AppCompatActivity {
    
    private static final String TAG = "Activity_viewDispatch";
    private Button btnClick;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        btnClick = findViewById(R.id.btn_click);
        btnClick.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                Log.i(TAG, "onClick: ");                
            }
        });
        
        btnClick.setOnTouchListener(new View.OnTouchListener() {
            @Override
            public boolean onTouch(View v, MotionEvent event) {
                switch (event.getAction()) {
                    case MotionEvent.ACTION_DOWN:
                        Log.i(TAG, "onTouch: ");
                        break;
                }
                //这里暂时先返回false查看日志
                return false; // 返回false，onTouchEvent会被调用
            }
        });
    }    
}
复制代码
```

当按钮的`onTouch()`方法的返回值为`false`时，打印的log日志为：

```
MyButton_viewDispatch: dispatchTouchEvent: 
Activity_viewDispatch: onTouch: 
MyButton_viewDispatch: onTouchEvent: ACTION_DOWN
MyButton_viewDispatch: dispatchTouchEvent: 
MyButton_viewDispatch: dispatchTouchEvent: 
MyButton_viewDispatch: dispatchTouchEvent: 
MyButton_viewDispatch: dispatchTouchEvent: 
MyButton_viewDispatch: dispatchTouchEvent: 
Activity_viewDispatch: onClick: 
复制代码
```

当按钮的`onTouch()`方法的返回值为`true`时，打印的log日志为：

```
MyButton_viewDispatch: dispatchTouchEvent: 
Activity_viewDispatch: onTouch: 
MyButton_viewDispatch: dispatchTouchEvent: 
MyButton_viewDispatch: dispatchTouchEvent: 
MyButton_viewDispatch: dispatchTouchEvent: 
MyButton_viewDispatch: dispatchTouchEvent: 
MyButton_viewDispatch: dispatchTouchEvent: 
复制代码
```

可以看到当`View`的`onTouch()`方法返回`false`时，`View`的`onTouchEvent()`方法和`onClick()`方法会被调用，当返回`true`时，这两个方法都不会被调用。这是什么原因呢？

## 8.2 源码解析

**目的：**

1、想得知为何`View`的`onTouch()`返回`false`时，它的`onTouchEvent()`和`onClick()`方法会被调用，而返回`false`时都不会被调用。

我们看`View`的`dispatchTouchEvent()`方法源码

```
public boolean dispatchTouchEvent(MotionEvent event) {
	......
    if (onFilterTouchEventForSecurity(event)) {
        if ((mViewFlags & ENABLED_MASK) == ENABLED && handleScrollBarDragging(event)) {
            result = true;
        }
        //noinspection SimplifiableIfStatement
        ListenerInfo li = mListenerInfo;
        if (li != null && li.mOnTouchListener != null
            && (mViewFlags & ENABLED_MASK) == ENABLED
            && li.mOnTouchListener.onTouch(this, event)) {
            result = true;
        }
        
        if (!result && onTouchEvent(event)) {
            result = true;
        }
    }
    ......
}
复制代码
```

`(mViewFlags & ENABLED_MASK) == ENABLED`代表控件`enable`，`li.mOnTouchListener`代表其设置的`OnTouchListener`，当我们为`View`通过`setOnTouchListener()`方法设置`touch`监听事件时，`li.mOnTouchListener`就不为空。`li.mOnTouchListener.onTouch(this, event)`代表`onTouch()`方法的返回值。

所以说当我们设置了`onTouch`监听事件并返回`false`时，源码这里的`result=false`，所以`if(!result&&onTouchEvent(event))`内的`onTouchEvent`方法会被调用。

当`onTouch()`返回`true`时，`if(!result&&onTouchEvent(event))内!result==false`，所以后面的`onTouchEvent()`方法不会被调用。

`onTouchEvent()`不被调用的时候，`onClick()`也不会被调用，他俩可能有关系，我们看下`onTouchEvent()`的源码

```
public boolean onTouchEvent(MotionEvent event) {
    
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
    
    if (clickable || (viewFlags & TOOLTIP) == TOOLTIP) {
        switch (action) {
            case MotionEvent.ACTION_UP:
                ......
                if ((mPrivateFlags & PFLAG_PRESSED) != 0 || prepressed) {  
                    ......
                    if (!mHasPerformedLongPress && !mIgnoreNextUpEvent) {
                        removeLongPressCallback();
                        if (!focusTaken) {
                            if (mPerformClick == null) {
                                mPerformClick = new PerformClick();
                            }
                            if (!post(mPerformClick)) {
                                performClickInternal();
                            }
                        }                    
                    }                               
			break;        
		return true;
	}        
    return false;            
}
复制代码
```

由其中的

```
final boolean clickable = ((viewFlags & CLICKABLE) == CLICKABLE
	|| (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE)
	|| (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE;
复制代码
```

可以看到，只要`View`的`clickable`和`longclickable`有一个为`true`，那么`clickable`就会为`true`。然后会进入到`switch`语句中，在经过各种判断后会执行到`performClickInternal()`方法，而该方法源码为以下内容

```
private boolean performClickInternal() {    
    notifyAutofillManagerOnClick();
    return performClick();
}
复制代码
```

可以看到调用了`performClick()`方法，接下来看它的源码

```
public boolean performClick() {
       
    notifyAutofillManagerOnClick();    
    final boolean result;
    final ListenerInfo li = mListenerInfo;
    if (li != null && li.mOnClickListener != null) {
        playSoundEffect(SoundEffectConstants.CLICK);
        li.mOnClickListener.onClick(this);
        result = true;
    } else {
        result = false;
    }
    sendAccessibilityEvent(AccessibilityEvent.TYPE_VIEW_CLICKED);   
    notifyEnterOrExitForAutoFillIfNeeded(true);    
    return result;
}
复制代码
```

可以看到`li.mOnClickListener.onClick(this);`，调用了`click`方法，所以说`onCLick()`方法在`onTouchEvent()`方法内被调用，`onTouchEvent`不被执行，那么`onCLick`一定不会执行。

## 8.3 小结：View分发的流程图



![img](https://user-gold-cdn.xitu.io/2019/7/29/16c3b7d7e6c50648?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



# 9. 一个U形图解释



![img](https://user-gold-cdn.xitu.io/2019/7/29/16c3b7dd0d1ea8de?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



# 10. 总结

其实如果只是想逻辑的话也很好理解。

`dispatchTouchEvent`代表分发事件，`onInterceptTouchEvent()`代表拦截事件，`onTouchEvent()`代表消耗事件，由自己处理。

默认状态下事件是按照从`Activity`到`ViewGroup`再到`View`的顺序进行分发的，分发下去处不处理是另一回事，分发完成后，不处理则向上一层回调，调用上一层的`onTouchEvent`进行处理事件，若`onTouchEvent`返回`true`，则表示在该层消耗了事件，若返回`false`，表示事件还没被处理，需要再向上回调一层，调用上一层的`onTouchEvent`方法。

以上就是全部内容，第一次分析源码，有错误的地方还望指出，参考文章大神写得也很详细，一定要看看。

多说一句，看源码确实很头大，尤其在看不懂却还要看到这么多文字就更耐不下心来了，但是当我看到一篇文章下的一个评论时，确实是激励了我，不懂就多读，没有解决不了的问题！


转载：[掘金-重拾丢却的梦](https://juejin.im/post/6844903901712351240)
