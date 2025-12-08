---
layout: post
title: 移动端重构遇到的坑（长期更新）
date: 2016-11-04 00:47:34
tags: 重构
categories:
 - UX
---

这篇文章用来记录平时移动端重构遇到的坑，以备忘，长期更新。
<!-- more -->
## 键盘遮挡影响布局
坑：ios下fixed属性失效。在编辑框输入内容时会弹出软键盘，而手机屏幕区域有限往往会遮住输入界面。
如下面截图，图片来自[%幻#影% - 博客园](https://www.cnblogs.com/cmblogs/p/4448336.html)
![键盘遮挡影响布局](/assets/img/2016/11/experience-1-2.jpg)
fix：
 - 方法一：css 设置flex:1
``` css
.container { /* 父类 */
    display: flex;
}
.container-content { /* 不想被遮挡的内容所在类 */
    flex: 1;
}
```
 - 方法二：js 监听输入框focus事件，聚焦时上移页面，失焦时恢复原状
``` js
$('input').on('focus', function() {
    $('.container-content').css('transform','translateY(-50%)');
})
$('.container-content').on('click', function() {
    $('.container-content').css('transform','translateY(0)');
})
```

## 在微信等webview中无法修改document.title
坑：会碰到需要动态修改document.title的需求，这时会遇到在微信等webview中无法修改document.title的坑
fix：
``` js
// hack在微信等webview中无法修改document.title的情况
var $iframe = $('<iframe src="#"></iframe>'),
    $body = $('body');
$iframe.on('load',function() {
    setTimeout(function() {
        $iframe.off('load').remove();
    }, 0);
}).appendTo($body);
```
``` css
iframe {
  visibility: hidden;
  width: 1px;
  height: 1px;
}
```

## 图片边缘出现截断情况
坑：在plus机上出现图片边缘截断情况。图标像素是奇数，前辈说移动端的图标都要求是偶数才不会出现问题。
![图片边缘截断](/assets/img/2016/11/experience-1-1.jpeg)
fix：编辑图像大小，改成偶数咯
P.S.也有可能是用了rem的缘故。较小的背景图（比如一些 icon）的 background-size 不要使用具体 rem 数值，裁剪后会出现边缘丢失。应使用与元素等尺寸切图，设定 background-size: contain|cover 来缩放。

## 安卓机丢掉rem小数部分
坑：IOS对小数的像素很敏感但是Android就不是，一些安卓机会丢掉rem小数部分
fix：
 - 根元素设成50px，以50px为基准，即好算，而且小数位会少一点。
 - 残暴点的就不用rem，用px，再想其他的自适应方案。

## input[type="file"]的change事件只调用一次
调用一次之后发现输入框没有绑定change事件，所以在回调函数内再绑定一次change事件。
``` js
$('input[type="file"]').on('change', function(e) {
  e.preventDefault();
  e.stopPropagation();
  var $target = $(e.target);
  var fileList = e.target.files||e.dataTransfer.files;
  // do something
  $target.replaceWith('<input type="file" class="fd-file" data-fd="file" id="file"'+Math.random()+'>');
  $('input[type="file"').on('change', function(e) {
    var $target = $(e.target);
    var fileList = e.target.files||e.dataTransfer.files;
    // do something
  })
})
```

## 参考
 - [用document.title=“xxx”动态修改title，在ios的微信下面不生效](https://segmentfault.com/q/1010000002926291)
