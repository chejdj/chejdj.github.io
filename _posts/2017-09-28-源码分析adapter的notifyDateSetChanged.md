---
layout: post
title: 源码分析adapter的notifyDataSetChanged
date: 2017-09-28 18:26:51
categories: 
- Android
tags:
- Android
- 源码分析
---


我们在使用listview控件的时候，总是会因为数据的改变，而需要更新listview控件的内容，这时候总是会调用adapter的notifyDataSetChanged（）方法，现在分析一下，调用这个方法具体实现了什么步骤。


### 第一步
首先调用了mDataSetobservable.notifyChanged()   （DataSetObservable类）方法
![这里写图片描述](http://img.blog.csdn.net/20170928180413715?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXFfMzI1NjU1NzU=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
### 第二步
接着 循环调用了 在DataSetObservable类中     mObservers.onChanged()方法 ，  mObservers是什么东西呢？？因为DataSetObservable继承Observable<T>这个类， 我们去看看！！
![这里写图片描述](http://img.blog.csdn.net/20170928180608853?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXFfMzI1NjU1NzU=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)


可以看到  mObserve就是一个List集合
![这里写图片描述](http://img.blog.csdn.net/20170928180935900?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXFfMzI1NjU1NzU=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

![这里写图片描述](http://img.blog.csdn.net/20170928181541786?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXFfMzI1NjU1NzU=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

### 第三步
可是，由前面可知道调用mObserver.change()方法，就是调用DataSetObservable里面的onchange()方法，可是我进去看却没有里面什么都没有实现，可见在别的地方重写了该方法，我们在setAdapter里面找到了，下面遮住的是就是注册
![这里写图片描述](http://img.blog.csdn.net/20170928182155903?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXFfMzI1NjU1NzU=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

AdapterDataSetObserver就是继承了DataSetObservable类在里面重写了onchange()函数

```
  @Override
        public void onChanged() {
            mDataChanged = true;
            mOldItemCount = mItemCount;
            mItemCount = getAdapter().getCount();

            // Detect the case where a cursor that was //previously invalidated has
            // been repopulated with new data.
 if (AdapterView.this.getAdapter().hasStableIds() && mInstanceState != null
                    && mOldItemCount == 0 && mItemCount > 0)
         {
                AdapterView.this.onRestoreInstanceState(mInstanceState);
                mInstanceState = null;
            } else {
                rememberSyncState();
            }
            checkFocus();
            //会导致调用measure()过程 和 layout()过程
            requestLayout();
        }

```
requestLayout()进行布局和重绘。
我们可以简单梳理一下更新的过程： listview通知刷新，首先实现一个DataSetObserver类，重写里面的onChanged回调方法，然后把这个对象添加（注册）到ArrayList中，这样当我们调用notifyDataSetChanged的时候，它会遍历这个ArrayList取出DataSetObserver对象（正常来说就一个，只调用一次setAdapter）,回调onChanged方法。onChanged里面就的requestLayout就实现重新绘制