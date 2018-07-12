---
title: LinearLayout android:layout_weight属性用法总结
---

&emsp;&emsp;`LinearLayout`，很常用的一种布局，当在使用这种布局方式时，为了达到较好的屏幕适配效果，可以选择使用`android:layout_weight`属性。当为布局中的每个组件指定了大小和权重之后，我们的android系统如何计算各组件实际所占空间呢？
<!--more-->
贴个自己常用的计算方法：
**实际大小=指定大小+（屏幕大小-（所有组件大小的和））\*权重比例。**

用一屏幕的大小减去所有组件大小的和，得到剩余大小，或理解为可分配空间，该值可为负。然后用我们给某个组件设定的值，加上其在剩余大小中按比例计算后的值，就是该组件最后的实际大小。
可用空间=屏幕大小-所有组件大小的和，
实际大小=指定大小+可用空间\*权重比例，
所以最后为：
实际大小=指定大小+（屏幕大小-（所有组件大小的和））*权重比例。

示例:
以`android:layout_width`为例，验证计算方式（屏幕宽度：L）
在`LinearLayout`中放置三个按钮，分别为Button1，Button2，Button3，`android:layout_height="wrap_content"`均指定为包裹内容，每个Button的`android:layout_weight`分别为1，2，3。

依据`android:layout_width`属性的设置，分三种情况：
1. `android:layout_width="0dp"`时：
组件大小=0+（L-（0+0+0））\*权重比例，组件大小和权重成正比。
各Button占的宽度：
Button1=0+(L-(0+0+0))\*(1/6)，为1/6L；
Button2=0+(L-(0+0+0))\*(2/6)，为2/6L；
Button3=0+(L-(0+0+0))\*(3/6)，为3/6L。
所以Button1:Button2:Button3 = 1:2:3。

    ![layout_width="0dp"][]

2. `android:layout_width="match_parent"`时：
组件大小=L+（L-（L+L+L））\*权重比例，组件大小和权重成反比。
如果前面组件已经占满屏幕，则剩余组件无法获得空间，不可见。
各Button占的宽度：
每个Button的`layout_width`指定为`match_parent`，即表示每个Button指定大小为父组件宽度，即L。
Button1=L+(L-(L+L+L))\*(1/6)，为4/6L；
Button2=L+(L-(L+L+L))\*(2/6)，为2/6L；
Button3=L+L(L-(L+L+L))\*(3/6)，为0。
所以Button1:Button2=2:1，Button3不可见。

    ![layout_width="match_parent"][]


3. `android:layout_width="wrap_content"`时：
组件大小=所需大小+(L-各组件所需大小和)\*权重比例，剩余大小即(L-各组件所需大小和)为正，则为正比，否则为反比。
该情况和组件内容占用大小有关，所以布局上并不是很明显的比例关系，因为比例关系仅指剩余大小的比例：
    剩余大小为正，即屏幕空间足够，所占空间和权重成正比。
    ![layout_width="wrap_content" enough][]

    剩余大小为负，即屏幕空间不足，所占空间和权重成反比，下面的图可能有点难看，希望想表达的意思到了。
    ![layout_width="wrap_content" less][]

`LinearLayout`的`android:layout_weight`属性用起来很简单，但是如果想深入理解其原理，就需要参考源代码了。

[layout_width="0dp"]: linearLayout/width_0.png "0 dp"
[layout_width="match_parent"]: linearLayout/width_match.png "match_parent"
[layout_width="wrap_content" enough]: linearLayout/width_enough.png "wrap_content enough"
[layout_width="wrap_content" less]: linearLayout/width_less.png "wrap_content less"
