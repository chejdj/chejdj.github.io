---
layout: post
title: ListView和RecycleView的区别
date: 2018-03-26 15:54:00
categories:
- Android
tags:
- Android
---
#### 一. getView()和onCreateViewHolder方法  
我想对于这个滑动控件来说，最神奇的地方就是他们的数据绑定函数，对于Listview控件来说就是它的适配器getView()函数

<!--more-->

<pre>
@Override
public View getView(int position,View convertview,ViewGroup parent){
  Fruit fruit=getItem(position);//获取Fruit对象，假设成员是Fruit
  View view;
  ViewHolder hodler;
  if(convertview==null){
  view =LayoutInflater.from(getContext().inflate(resoursceId,parent,false);
  holder=new ViewHolder();
  holder.text=view.findViewById...//元素绑定
  conterview.setTag(viewholder);
  else{
  view=convertview; 
  holder=(ViewHolder)conterview.getTag;
  }
  viewHolder.textview =.....//就是元素赋值

  }
}
</pre>  


对于listview来说，每一次滑入屏幕中，都要调用getView来展示内容，上面的实现还是经过优化之后的，利用convertview来缓存我们自定义的ViewHolder来避免每一次函数调用时，都要findViewByID来增加时间成本，而且这些都不是必须的，如果不做优化，那就非常的卡。  
下面来看看RecycleView的几个重要函数
```
public class collect_adapter extends RecyclerView.Adapter<collect_adapter.CollectHolder> {
  private ArrayList<String> mPoem;
  public collect_adapter(ArrayList<String> mPoem) {
  this.mPoem = mPoem;
  }

static class CollectHolder extends RecyclerView.ViewHolder {
  public TextView textview;
  public CollectHolder(View view) {
  super(view);
   this.textview = (TextView) view.findViewById(R.id.text_title);
        }
  }

@Override
public CollectHolder onCreateViewHolder(ViewGroup parent, int viewType) {
  View view = LayoutInflater.from(parent.getContext()).inflate(R.layout.list_item, parent, false);
   CollectHolder holder = new CollectHolder(view);
   return holder;
   }

@Override
public void onBindViewHolder(final CollectHolder holder, final int position) {
   String str = (String) mPoem.get(position);
   holder.textview.setText(str);
   }

@Override
 public int getItemCount() {
   return mPoem.size();
   }
}
``` 

首先他有自己的ViewHolder而且必须要实现，就不需要自己定义，解决了上面的ListView不适用ViewHolder问题  

#### 二. 布局  
listview只能在垂直方向上滚动，但是RecycleView确可以有很多布局  
<pre>
LinearLayoutManager layout = new LinearLayoutManager(this);
recyclerView.setLayoutManager(layout);
recyclerView.setAdapter(adapter);
</pre>
在使用RecyclieView需要传入布局形式，LayoutManager中制定了一套可以扩展的布局排列接口，所以我们也可以重写LayoutManager来定制自己需要的布局。  

#### 三.更新数据  
recycleView可以支持在添加，删除或者移动Item的时候，`RecyclerView.ItemAnimator`添加动画效果，而listview不支持,而且看网上博客，说RecycleView具有四重缓存，listview具有两重缓存机制ListView和RecyclerView最大的区别在于数据源改变时的缓存的处理逻辑
>> 在刷新数据的时候，ListView是"一锅端"，将所有的mActiveViews都移入了二级缓存mScrapViews，而RecyclerView则是更加灵活地对每个View修改标志位，区分是否重新bindView

上面的还有待考究，四重缓存，不知道在那里看出来的，源码暂时没有看到)，反正就是一个字快  

#### 四.控件点击事件  
listview使用`onItemClickListener`，而recyclerview则是使用`OnItemTouchListener`接口探测触摸事件，可以检测更多的行为，但是我没有看到获取position这个属性，实际自己用的时候，看了一下网上的教程自己定义[设置Item点击事件](https://blog.csdn.net/dmk877/article/details/50816933)  
