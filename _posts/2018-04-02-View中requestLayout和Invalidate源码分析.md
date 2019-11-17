---
layout: post
title: View中requestLayout和Invalidate源码分析
date: 2018-04-02 16:15:00
categories:
- Android
tags:
- Android
---


我们都知道requestLayout和Invalidate都会导致View的绘制，那他们到底，有什么区别呢？我们可以去View的源码看看究竟(因为没有看过相关源码，这里重新补一下，自己太菜了)  

<!--more-->  


#### requestLayout的源码分析  
首先我贴出，requestlayout的注释  
>>
     * Call this when something has changed which has invalidated the
     * layout of this view. This will schedule a layout pass of the view
     * tree. This should not be called while the view hierarchy is currently in a layout
     * pass ({@link #isInLayout()}. If layout is happening, the request may be honored at the
     * end of the current layout pass (and then layout will run again) or after the current
     * frame is drawn and the next layout occurs.
     *
     * <p>Subclasses which override this method should call the superclass method to
     * handle possible request-during-layout errors correctly  

当View发生改变使得这个view的布局无效的时候，调用该方法，如果View正在请求布局的时候，View树正在进行布局，那么requestlayout会等到布局流程完成之后，或则绘制流程完成且下一次布局出现的时候执行。  
下面看源码  
`if (mMeasureCache != null) mMeasureCache.clear();`  
mMeasureCache是属于LongSparseLongArray类型的，可以看到这是一种比使用HasMap更加高效来存储longs类型的容器
>> * SparseArray mapping longs to Objects.  Unlike a normal array of Objects,
 * there can be gaps in the indices.  It is intended to be more memory efficient
 * than using a HashMap to map Longs to Objects, both because it avoids
 * auto-boxing keys and its data structure doesn't rely on an extra entry object
 * for each mapping.  

```
if (mAttachInfo != null && mAttachInfo.mViewRequestingLayout == null) {
            // Only trigger request-during-layout logic if this is the view requesting it,
            // not the views in its parent hierarchy
            ViewRootImpl viewRoot = getViewRootImpl();
            if (viewRoot != null && viewRoot.isInLayout()) {
                if (!viewRoot.requestLayoutDuringLayout(this)) {
                    return;
                }
            }
            mAttachInfo.mViewRequestingLayout = this;
        }

```  
第一行中mAttachInfo是AttachInfo类型,它表示依附于父View窗口给子View的一些信息,一般都不为空  
>>  * A set of information given to a view when it is attached to its parent window.  
     
mAttachInfo.mViewRequestinglayout用于记录哪一个View发起了requestlayout信息，整个View源码就下面的`mAttachInfo.mViewRequestingLayout=this`赋值，默认开始为null  
>> Used to track which View originated a requestLayout() call, used when
   requestLayout() is called during layout.  

`if(viewRoot!=null && viewRoot.isInLayout())`判断当前这个View树是否在进行布局流程，如果在的话就调用requestLayoutoutDuringLayout(this)让这一次的布局延时进行。  
```
mPrivateFlags |= PFLAG_FORCE_LAYOUT;
mPrivateFlags |= PFLAG_INVALIDATED;
```  
设置mPrivateFlags的两个标志位，PFLAG_FORCE_LAYOUT,通过表面的意思可以知道这是一个布局标志位，就会执行View的mearsure()和layout()方法  
```
if (mParent != null && !mParent.isLayoutRequested()) {
            mParent.requestLayout();
        }
if (mAttachInfo != null && mAttachInfo.mViewRequestingLayout == this) {
            mAttachInfo.mViewRequestingLayout = null;
        }
```  
最后一个判断就是把之前的标记清除一下，最重要的是前面的一个判断，如果父容器不为空，并且父容器没有LayoutRequest就调用父容器的requestLayout,因为父容器是ViewGroup没有重写requestLayout，但是它的父类也是View就又会调用它父容器的requestLayout，不断上传并且为父容器设置**PFLAG_FORCE_LAYOUT**和**PFLAG_INVALIDATED**两个标志位，最后到顶级容器DecorView，但是DecorView的mparent是ViewRootImpl对象，它也设置两个标志位，然后就调用ViewRootImpl的requestLayout  
```
public void requestLayout() {
        if (!mHandlingLayoutInLayoutRequest) {
            checkThread();
            mLayoutRequested = true;
            scheduleTraversals();
        }
    }
```  
mHandlingLayoutInLayoutrequest是一个boolean类型，它会在performLayout中被设置为true,这里表示的意思就是当前并不处于Layout过程中  
```
void scheduleTraversals() {
        if (!mTraversalScheduled) {
            mTraversalScheduled = true;
            mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
            mChoreographer.postCallback(
                    Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
            if (!mUnbufferedInputDispatch) {
                scheduleConsumeBatchedInput();
            }
            notifyRendererOfFramePending();
            pokeDrawLockIfNeeded();
        }
}
```  
这里面的成员mChoregrapher是属于Choreographer类型,大致意思这个按照一帧一帧绘制。该方法对于绘制的请求经过了Choreographer的编排后，最终会调用回ViewRootImpl.doTraversal()方法，在doTraversal又会调用performTraversals()方法，这个代码太长了，相信知道View的工作原理的知道，performTraversals是View绘制流程的开始，它会经过measuer,layout,draw三个过程。所以看到这里，我们大概就知道了requestLayout会触发整个界面的绘制(measuer,layout,draw)(注意前面每一层设置的标志位很重要，因为它决定了界面绘制是否需要执行)  
>> The choreographer receives timing pulses (such as vertical synchronization)
 * from the display subsystem then schedules work to occur as part of rendering
 * the next display frame.   
 
```
public void layout(int l, int t, int r, int b) {
......
//判断标志位
if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
            onLayout(changed, l, t, r, b);
....
}

public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
  final boolean forceLayout = (mPrivateFlags & PFLAG_FORCE_LAYOUT) == PFLAG_FORCE_LAYOUT;
 //是否设置了 PFLAG_FORCE_LAYOUT标志位
 final boolean needsLayout = specChanged
                && (sAlwaysRemeasureExactly || !isSpecExactly || !matchesSpecSize);
 //一般这里为false,在View不变的情况下
   if (forceLayout || needsLayout) {//判断是否需要measuer 
}
}


public void draw(Canvas canvas) {
final boolean dirtyOpaque = (privateFlags & PFLAG_DIRTY_MASK) == PFLAG_DIRTY_OPAQUE &&
                (mAttachInfo == null || !mAttachInfo.mIgnoreDirtyState);

//如果设置了 PFLAG_DIRTY_MASK, &的结果不相等，返回false，所以就绘制
// PFLAG_DIRTY_OPAQE标志位表示该View是否已经被invalidate
if (!dirtyOpaque) {  
   drawBackground(canvas);//后面还有判断就一一写了
}

}


```
**总结一下**调用requestlayout，会导致自己的View设置两个标志位，并且向上调用父类的requestlayout并且设置标志位，不断向上传递，最后上传到ViewRootImp执行performTraversals执行measure,layout(在三大流程中检查标志位)（更新一下，不一定执行onDraw,没有设置PFLAG_DIRTY_MASK标志位）  

#### Invalidate源码分析
我们来看一下View的invalidate调用
```
public void invalidate() {
        invalidate(true);
}
...

public void invalidate(boolean invalidateCache) {
  invalidateInternal(0, 0, mRight - mLeft, mBottom - mTop, invalidateCache, true);
}
....

void invalidateInternal(int l, int t, int r, int b, boolean invalidateCache,
            boolean fullInvalidate) {
...

if (skipInvalidate()) {//判断该View是否可见 或则是否在动画中
            return;
}
//判断View的一些标志位，判断View是否需要重新绘制
if ((mPrivateFlags & (PFLAG_DRAWN | PFLAG_HAS_BOUNDS)) == (PFLAG_DRAWN | PFLAG_HAS_BOUNDS)
                || (invalidateCache && (mPrivateFlags & PFLAG_DRAWING_CACHE_VALID) == PFLAG_DRAWING_CACHE_VALID)
                || (mPrivateFlags & PFLAG_INVALIDATED) != PFLAG_INVALIDATED
                || (fullInvalidate && isOpaque() != mLastIsOpaque)) {
... //里面执行绘制方法
      mPrivateFlags |= PFLAG_DIRTY; //给这个View设置PFLAG_DIRTY标志位
}
...
// Propagate the damage rectangle to the parent view.
            final AttachInfo ai = mAttachInfo;
            final ViewParent p = mParent;
            if (p != null && ai != null && l < r && t < b) {
                final Rect damage = ai.mTmpInvalRect;
                damage.set(l, t, r, b);//l,t,r,b是需要重新绘制的View的范围
                p.invalidateChild(this, damage);//p是ViewParent类型，向父类传递绘制事件
}
}
....


public final void invalidateChild(View child, final Rect dirty) { //dirty是需要绘制的区域

....//内部存在一个循环

do {
                View view = null;
                if (parent instanceof View) {
                    view = (View) parent;
                }

                if (drawAnimation) {
                    if (view != null) {
                        view.mPrivateFlags |= PFLAG_DRAW_ANIMATION;
                    } else if (parent instanceof ViewRootImpl) {
                        ((ViewRootImpl) parent).mIsAnimating = true;
                    }
                }

                // If the parent is dirty opaque or not dirty, mark it dirty with the opaque
                // flag coming from the child that initiated the invalidate
                if (view != null) {
                    if ((view.mViewFlags & FADING_EDGE_MASK) != 0 &&
                            view.getSolidColor() == 0) {
                        opaqueFlag = PFLAG_DIRTY;
                    }
                    if ((view.mPrivateFlags & PFLAG_DIRTY_MASK) != PFLAG_DIRTY) {
                        view.mPrivateFlags = (view.mPrivateFlags & ~PFLAG_DIRTY_MASK) | opaqueFlag;
                    }
                }
                //循环调用parent.parent的invalidateChildInparent方法，向上传递
               //最终还是会传递到RootViewImp类中
                parent = parent.invalidateChildInParent(location, dirty);
                if (view != null) {
                    // Account for transform on current parent
                    Matrix m = view.getMatrix();
                    if (!m.isIdentity()) {
                        RectF boundingRect = attachInfo.mTmpTransformRect;
                        boundingRect.set(dirty);
                        m.mapRect(boundingRect);
                        dirty.set((int) Math.floor(boundingRect.left),
                                (int) Math.floor(boundingRect.top),
                                (int) Math.ceil(boundingRect.right),
                                (int) Math.ceil(boundingRect.bottom));
                    }
                }
            } while (parent != null);
}
...

public ViewParent invalidateChildInParent(final int[] location, final Rect dirty) {
    ....
   dirty.offset(location[CHILD_LEFT_INDEX] - mScrollX,
                        location[CHILD_TOP_INDEX] - mScrollY);
    ....
   dirty.union(0, 0, mRight - mLeft, mBottom - mTop);
    ....   //这个方法就是对于dirty区域做一些转换，变成父容器的dirty区域
}
```  
接下来我们看看ViewRootImp的invalidateChildInParent方法，它会触发`scheduleTraversals();`方法,然后执行onMeasure,onlayout,draw方法，但是它并没有设置**PFLAG_FORCE_LAYOUT**标志位，所以不执行，只执行draw方法。  
**总结一波**调用Invalidate方法给该View设置PFLAG_DIRTY标志位，然后不断计算在父容器中的区域向上传递(设置标志位)，最终传递到RootViewImp执行`scheduleTraversals`然后执行`performTraversals`，然后measure,layout,draw三个方法，由于设置的标志位，只能执行draw方法，前面两个不执行(不是不执行，只是不执行特有的方法)

 



