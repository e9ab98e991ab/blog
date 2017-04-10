# 关于Span的flags说明
##背景
很多时候希望，文本的不同部分有不同的表示方式，比如某些字符串为红色、某些字符串加粗、添加背景色等等。如图所示：
![](https://www.github.com/wslaimin/blog/raw/master/pics/span.JPG)
要实现这种效果，通常使用SpannableString类或SpannableStringBuilder类。两者都要用到setSpan(Object what,int start,int end,int flags)方法，关于最后一个参数的含义就是本文章的主题。
##flag的种类

 - Spanned.SPAN_INCLUSIVE_INCLUSIVE
 - Spanned.SPAN_EXCLUSIVE_EXCLUSIVE
 - Spanned.SPAN_INCLUSIVE_EXCLUSIVE
 - Spanned.SPAN_EXCLUSIVE_EXCLUSIVE

无非就是INCLUSIVE和EXCLUSIVE的排列组合。排除的是什么，包含的又是什么？

首先看个SpannableString的例子：

```java
SpannableString spannableString=new SpannableString("0123456");
spannableString.setSpan(new ForegroundColorSpan(Color.RED),1,5,
Spanned.SPAN_EXCLUSIVE_EXCLUSIVE);
mTextView.setText(spannableString);
```
 
 运行的效果如图：
 ![](https://www.github.com/wslaimin/blog/raw/master/pics/span.JPG)
 
 ><font color="0xff000000">注意：更改flag为其他值结果都是一样的</font>
 
 如你所见，<font color="0xff000000">flags对于不变字符串是没有任何意义的。</font>

再来看看SpannableStringBuilder的例子：

```java
SpannableStringBuilder builder=new SpannableStringBuilder("0123456");
ForegroundColorSpan foregroundColorSpan=new ForegroundColorSpan(Color.RED);
builder.setSpan(foregroundColorSpan,1,5,Spanned.SPAN_INCLUSIVE_INCLUSIVE);
builder.insert(5,"xx");
builder.insert(1,"xx");
mTextView.setText(builder);
```

运行效果如图：
![](https://www.github.com/wslaimin/blog/raw/master/pics/spanbuilder.JPG)

可以看到，索引为1和5的位置插入的"xx"都是红色的，说明flag起作用了，foregroundColorSpan的作用范围为[1,5]。

吧Spanned.SPAN_INCLUSIVE_INCLUSIVE换成Spanned.SPAN_EXCLUSIVE_INCLUSIVE的效果：
![](https://www.github.com/wslaimin/blog/raw/master/pics/spanbuilderex.JPG)

很显然foregroundColorSpan的作用范围为(1,5]。

如你所见，<font color="0xff000000">对于可变字符串flags决定了作用的区间。</font>

##总结

 1. flags只对可变字符串起作用，而且是在发生改变时起作用。
 2. 对于可变字符串，flags决定了作用的区间。