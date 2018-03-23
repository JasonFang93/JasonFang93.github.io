title: 我不知道的行高——line-height
date: 2016-05-07
category: 技术
tags: [CSS]
---

## 概述
+ 对于块级元素，CSS属性line-height指定了元素内部line-boxes的最小高度。
+ 对于非替代行内元素，line-height用于计算line box的高度。
+ 对于替代行内元素，如button 或其他input元素，line-height没有影响
<!-- more -->

## 取值
+ `normal`
取决于用户代理。桌面浏览器（包括火狐浏览器）使用默认值，约为1.2，这取决于元素的 font-family。(存在差异性)
+ `<number>`
所用的值是无单位数值<number>乘以元素的 font size。计算出来的值与使用数值指定的一样。大多数情况下，使用这种方法设置line-height是首选方法，在继承情况下不会有异常的值。    
+ `<length>`
指定<length>  用于计算 line box 的高度。查看<length> 获取可能的单位。
+ `<percentage>`
与元素自身的字体大小有关。计算出的值是给定的百分比值乘以元素计算出的字体大小。
+  `inherit`

缩写：line-height的值紧跟font-size的值，使用/分开（eg: font: 16px/normal arial;）

## 计算
+ 百分比
所有继承下来的元素会忽略本身的font-size，而使用相同的、计算出来的line-height；line-height不会随着相关的font-size做相应的比例缩放，可能导致有的地方太紧，而有的地方太松（缺点）。

|element| font-size | line-height | 计算后的line-height|
|------- |----------| ------------|--------------------|
|body    |16px      |  120%       |16px * 120% = 19.2px|
|h1      |32px      |继承的计算后的值|           19.2px|
|p       |16px      |继承的计算后的值|           19.2px|
|#footer |12px      |继承的计算后的值|           19.2px|

+ 长度
所有继承下来的元素会忽略本身的font-size，而使用相同的继承来的line-height;也不会随着相关的font-size做相应的比例缩放。

|element| font-size | line-height | 计算后的line-height|
|------- |----------| ------------|--------------------|
|body    |16px      |  20px       |          20px      |
|h1      |32px      |继承的计算后的值|        20px     |
|p       |16px      |继承的计算后的值|        20px     |
|#footer |12px      |继承的计算后的值|        20px     |

+ normal（约为1.2）
所有继承下来的元素**不会**忽略本身的font-size，使用基于font-size算来的line-height;也不会随着相关的font-size做相应的比例缩放。现在会做相应缩放

|element| font-size | line-height | 计算后的line-height|
|------- |----------| ------------|--------------------|
|body    |16px      |  normal     | 16px * 1.2（约）= 19.2px|
|h1      |32px      |  normal     | 32px * 1.2（约）= 38.4px|
|p       |16px      |  normal     | 16px * 1.2（约）= 19.2px|
|#footer |12px      |  normal     | 11.2px * 1.2（约）= 13.44px|

+ 纯数字（推荐）
所有继承下来的元素使用基于font-size算来的line-height;会随font-size做相应缩放。（偶尔标题会有的地方比较松）

|element| font-size | line-height | 计算后的line-height|
|------- |----------| ------------|--------------------|
|body    |16px      |  1.5        | 16px * 1.5 = 24px|
|h1      |32px      |  系数：1.5  | 32px * 1.5 = 48px|
|p       |16px      |  系数：1.5  | 16px * 1.5 = 24px|
|#footer |12px      |  系数：1.5  | 12px * 1.5 = 18px|
*可以考虑内容设置line-height：1.5；标题设置1.2

## 四种盒子
containing box
inline boxes
line boxes（inline boxes在containing boxes中一个接一个形成line boxes，高取决于内部最高的inline box）
content area（围绕文字、看不见的box，高取决于font-size）

> 参考：http://www.slideshare.net/daemao/line-height-2470819

## 应用
+ 单行文字居中
一般在做单行文字居中都是采用该策略——将line-height设置与height等高；但是height是不需要设置的
+ 固定高度的多行文字居中
+ 图片垂直居中
