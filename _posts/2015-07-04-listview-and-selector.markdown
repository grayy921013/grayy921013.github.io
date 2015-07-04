---
title: 实习收获笔记
layout: post
guid: urn:uuid:E02F7E4F-A8A5-452C-9429-4312025C508E
tags:
  - android
  - listview
  - selector
summary:
   最近实习过程中的一些收获
---

#关于listview
之前做项目的时候对性能优化的考虑比较少，总是图写的快写的方便，所以养成了一些坏习惯。最近实习中意识到了之前在listview使用上的一些问题。  
##viewholder的使用
viewholder可以防止每次getview的时候再调用一大堆getviewbyid()，节约性能。  
定义viewholder：
{% highlight JAVA %}
static class ViewHolder {
  TextView text;
  TextView timestamp;
  ImageView icon;
  ProgressBar progress;
  int position;
}
{% endhighlight %}
如果convertview为空则inflate并将创建好的viewholder对象设置为view的tag；如果不为空则用gettag获取viewholder对象
{% highlight JAVA %}
ViewHolder holder = new ViewHolder();
holder.icon = (ImageView) convertView.findViewById(R.id.listitem_image);
holder.text = (TextView) convertView.findViewById(R.id.listitem_text);
holder.timestamp = (TextView) convertView.findViewById(R.id.listitem_timestamp);
holder.progress = (ProgressBar) convertView.findViewById(R.id.progress_spinner);
convertView.setTag(holder);
{% endhighlight %}
##model解耦合
将model相关的逻辑都放到model里面：比如相关的attribute；以及对各种时间的响应，如onclick、onlongclick等。  
以前会吧所有的逻辑放在adapter里面，而adapter又作为fragment或者activity的内部类，并在setonclicklistener中新建匿名类。  
这样会对性能和日后维护带来很多坏处：首先adapter会显得非常臃肿，各种不同种类的cell的所有逻辑都会塞在里面；第二内部类会消耗比较多的内存空间；第三在getview中创建对象会使滑动listview创建大量的对象，并给gc带来很大的压力。

#关于selector
关于selector有一个小的发现。之前想要实现一个简单的按下效果，所以创建了一个selector，然而效果怎么样都无法实现。最后发现与item的顺序有关。  
我的xml文件如下
{% highlight xml %}
<item android:drawable="@drawable/background_normal">
<item android:state_pressed="true" android:drawable="@drawable/background_pressed">
{% endhighlight %}
问题在于如果在前面的item匹配成功就不会继续匹配后面的item了，而我这种写法第一个item必然匹配成功。正确的写法是讲默认的item放在最后
{% highlight xml %}
<item android:state_pressed="true" android:drawable="@drawable/background_pressed">
<item android:drawable="@drawable/background_normal">
{% endhighlight %}
或是将默认item加上state
{% highlight xml %}
<item android:state_pressed="false" android:drawable="@drawable/background_normal">
<item android:state_pressed="true" android:drawable="@drawable/background_pressed">
{% endhighlight %}