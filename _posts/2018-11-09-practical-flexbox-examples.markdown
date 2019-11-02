---
layout: post
title: "Practical flexbox examples"
date: 2018-11-09
tags: css flexbox html
---

<p class="intro"><span class="dropcap">T</span>he Flexbox CSS styling of web sites is gaining more and more popularity. This post is not a complete overview of what you can do with flexbox but rather a gathering of different flexbox techniques that I've found useful over the years.</p>

In the initial examples below I'll assume the following HTML snippet as a base.

{% highlight html linenos %}
<div class="sectionWrapper">
    <div class="sectionContent">Section longer 1</div>
    <div class="sectionContent">Section 2</div>
    <div class="sectionContent">S 3</div>
    <div class="sectionContent">Section 4</div>
</div>
{% endhighlight %}

#### Basic flexbox layout

Use `flex-direction` to swap between row and column. This indicates direction of items.

{% highlight csharp linenos %}
.sectionWrapper {
    display: flex;
    flex-direction: row;
}
{% endhighlight %}

{%
  include image.html
  url="2018-11-09-flexbox/divs-in-columns.png"
  alt=""
  description=""
%}

If you line up container divs (i.e. rows in a grid) then you need to think to opposite - the attribute `column` means your container becomes rows, one of top of the other, in a column.

{%
  include image.html
  url="2018-11-09-flexbox/divs-in-rows.png"
  alt=""
  description=""
%}

#### Fill space

As seen in the previous example, the divs are not filling up the whole row. This can be solved by increasing the space between the items using `justify-content`.

{% highlight css linenos %}
.sectionWrapper {
    display: flex;
    justify-content: space-between;
}
{% endhighlight %}

The result would look like this

{%
  include image.html
  url="2018-11-09-flexbox/divs-on-row-space-between.png"
  alt=""
  description=""
%}

Other options instead of `space-between` are `space-evenly`, `space-around`, `flex-start` and `flex-end`

Another way to fill the row is to increase the size of the divs which can be done by adding the `flex-grow` property.

{% highlight css linenos %}
.sectionContent {
    flex-grow: 1
}
{% endhighlight %}

The divs are now distributed all over the row.

{%
  include image.html
  url="2018-11-09-flexbox/divs-on-row-fill-space.png"
  alt=""
  description=""
%}

In a practical example we simulate four lists with drag-and-drop behaviour where the purple sections show the droppable areas. However, by default the divs only fill up the area where you have items inside them, which can make a column like S2 difficult to drop items into.

{%
  include image.html
  url="2018-11-09-flexbox/items-in-dnd-lists.png"
  alt=""
  description=""
%}

By adding `flex-grow: 1` for the purple droppable div, we solve this problem.

{%
  include image.html
  url="2018-11-09-flexbox/items-in-dnd-lists-same-height.png"
  alt=""
  description=""
%}

The same technique can be used to push a footer at the bottom of a page, or a card. Simply let the middle content of the item have `flex-grow:1` and the footer will be pushed down to the bottom

{%
  include image.html
  url="2018-11-09-flexbox/flexbox-footer-pushed-down2.png"
  alt=""
  description=""
%}

#### Fill space with same size for all divs

By default, `flex-basis` is set to auto to adjust for the different size of the elements inside, as can be seen here. 

{%
  include image.html
  url="2018-11-09-flexbox/divs-on-row-fill-space.png"
  alt=""
  description=""
%}

To make all columns have the same width we need to set the `flex-basis`, forcing all columns to have the same base.

{% highlight css linenos %}
.sectionContent {
    flex-grow: 1;
    flex-basis: 0;
}
{% endhighlight %}

The result will then look like this

{%
  include image.html
  url="2018-11-09-flexbox/sections-with-same-size.png"
  alt=""
  description=""
%}

#### Align content in center

Horizontal alignment has always been easy through auto margin. But vertical alignment has been a pain and always forced me to google it. With flex box style it's so much easier.

{% highlight css linenos %}
.centerContent {
    display: flex;
    align-items: center;
    justify-content: center;
}
{% endhighlight %}

This is how the titles (and other parts) in the above examples got centered in the middle of the divs

#### Separate items

Normally, the divs are separated by the same amount of space. But by tweaking the margin, you can change that behaviour.

{% highlight css linenos %}
.sectionWrapper {
    display: flex;
    flex-direction: row;
}
.sectionContent {
    margin: 10px;
}
.sectionContent:last-child {
    margin-left: auto;
}
{% endhighlight %}

{%
  include image.html
  url="2018-11-09-flexbox/divs-on-row-last-to-right.png"
  alt=""
  description=""
%}

#### Divs overflowing on next row

By default, when `flex-direction` is set to row then all items are displayed on a single row. To wrap items on several rows, use the `flex-wrap` setting.

{% highlight css linenos %}
.overflowingList{
    display: flex;
    flex-direction: row;
    flex-wrap: wrap;
}
{% endhighlight %}

{%
  include image.html
  url="2018-11-09-flexbox/flex-wrap.png"
  alt=""
  description=""
%}

#### Any neat Flexbox snippets you think I should add?
Do you have any good suggestions of useful Flexbox examples that you think I should have added? Please write them to the comments below.