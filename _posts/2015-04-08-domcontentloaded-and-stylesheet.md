---
layout: post
title:  "翻译-DOMContentLoaded和样式表"
categories: Javascript
---

DOMContentLoaded是现代Javascript用途中重要的一部分。在HTML代码被从服务器完全获取，完整的DOM树被创建，并且脚本可以通过DOM API操作DOM树时，DOMContentLoaded事件会被触发。通常，这个时间点被成为"DOM Ready"。与load事件相比，图片，iframe，插件等资源不会阻塞DOMContentLoaded事件。因此，DOMContentLoaded事件对希望尽早给页面绑定Javascript行为的开发者是一个理想的选择。

几乎所有Javascript框架都允许使用者在"on DOM ready"的时候执行他们的脚本。将代码定义在一个私有的function scope中并在"on DOM ready"的时候执行是一个常用的Javascript编程模式。在这样的启动方法中，可借助于选择器引擎查询DOM。例如，获得了DOM的信息，给元素绑定监听事件，修改DOM。正如我所说，这个模式对于DOM脚本编程非常重要。

我们在著名的框架中是如何使用DOMContentLoaded事件的呢：

```javascript
//jQuery:
jQuery(document).ready(function($) {});
jQuery(function($) {});

//Prototype:
document.observe('dom:loaded', function() {});

//Mootools:
window.addEvent('domready', function() {
});
```

IE8和IE8以下的IE浏览器不支持DOMContentLoaded事件，所以这些框架通过使用[doScroll()的实现方式][1]

## DOMContentLoaded和外部样式表

为了在我的Javascript教程中准确地描述DOMContentLoaded事件的行为，我进行了很多测试。最主要的问题是：外部样式表的加载是否会阻塞DOMContentLoaded事件？这个问题不是最新的，它已经被研究了很多年。

对于大部分脚本，在样式表加载完成后再执行它们是有道理的，因为这些脚本依赖于应用到DOM上的CSS规则。例如，对于某些脚本的初始化，它们依赖于元素的位置，大小。

[Mozilla][2]中定义到，DOMContentLoaded会在文档被解析，并且里面的脚本被执行后触发。但实际上，DOMContentLoaded有时候会根据脚本的位置和类型，被样式表的加载影响。

## 使用外部脚本在Gecko和Webkit中阻塞DOMContentLoaded

1. 测试用例#1：样式表后无脚本
如果样式表之后没有脚本，那么DOMContentLoaded不会被阻塞。这对所有支持DOMContentLoaded的浏览器都是成立的。

2. 测试用例#2：样式表后有外部脚本
There are several exceptions in which Gecko and Webkit do wait for the stylesheet to load before DOMContentLoaded fires. 。最常见的案例是`<link rel="stylesheet">`后面跟着一个外部脚本`<script src="..."></script>`。这个脚本可以放在文档的任意部分，只要它在`<link>`元素后面。

测试用例：
html:
```html
<!DOCTYPE html>
<head>
<link rel="stylesheet" href="http://molily.de/weblog/stylesheet.css">
<script src="http://molily.de/weblog/script.js"></script>
</head>
<body>
<div id="element">The element</div>
</body>
```

stylesheet.css:
```css
#element { color: red; }
```

script.js:
```javascript
document.addEventListener('DOMContentLoaded', function () {
    // should read #FF0000 or rgb(255, 0, 0)
    alert(getComputedStyle(document.getElementById('element'), null).color);
}, false);
```

为了论证浏览器之间的差异，我强制给HTTP服务器给样式表添加了3秒的延迟，这样文档就可以在样式表被完全接受到前解析完毕。

上面的代码在Firefox，Safari和Chrome中的行为都如我所预期，但是在Opera中不同。

在样式表后放置外部脚本是一个常见的做法。当你希望在脚本中读写样式已经生效的DOM时，[jQuery文档][3]推荐这种编写元素的顺序。即使这个样式表需要10秒来加载，并且文档在1秒内就被获取并解析完毕，DOMContentLoaded事件也会等到样式表加载并解析完毕后才触发。

这里涵盖了DOM脚本编程的优缺点。你可以依赖于样式表，但你的脚本必须一直等待，直到样式表加载完毕，才能遍历DOM数并添加监听事件。

## 背景知识：HTML解析
这个观测基于更底层的浏览器怪癖-HTML解析和脚本执行算法。

### Gekco, Webkit和IE中，样式表会阻塞外部脚本的执行
在这些浏览器中，**样式表的加载会阻塞外部脚本的执行。** 测试用例#2的文档head中包含了以下标记：

```html
<link rel="stylesheet" href="http://molily.de/weblog/stylesheet.css">
<script src="http://molily.de/weblog/script.js"></script>
```

当前的Gecko和Webkit版本，和IE8会并行下载样式表和脚本。但是不会执行脚本，直到样式表被加载完毕。并且同时它们不会渲染页面。你可以通过使用Firebug的网络标签，Safari Webjianchaqi 的Resources标签，或者常用的Chrome开发者工具的Timeline标签来验证这一点。
![chrome-timeline][4]
由于DOMContentLoaded会在所有脚本都执行完毕后才触发，因此DOMContentLoaded事件会在样式表被加载并解析后才触发。

对于IE8也是如此，当然，除了DOMContentLoaded事件。

## 在Opera中，样式表不会阻塞外部脚本的执行

但是，这个延迟DOMContentLoaded的方法在Opera中不生效。Opera一加载完脚本后就会开始解析。这导致了增加的渲染，提高了感知的性能，但是会出现从无样式变换到有样式的闪烁。

为了兼容Opera的差异，jQuery1.2.1到1.2.6在DOMContentLoaded时间后进行了额外的检查。在这些jQuery版本中，它保证DOM ready事件触发时，所有样式表都已经被加载并解析了。jQuery 1.3抛弃了[这个方案][5]。（据我所知，Prototype和Mootools完全忽略了这个问题。）

## 在Gecko和IE中，样式表会阻塞内部脚本的执行
内部脚本也可以引起不同的浏览器行为：

测试用例#3
```html
<link rel="stylesheet" href="http://molily.de/weblog/stylesheet.css">
<script> Some inline JavaScript code </script>
```

在IE和Gecko中，样式表会阻塞随后内部脚本的执行。因此，DOMContentLoaded事件被延迟了。

在Webkit和Opera中，内部脚本会马上执行。因此，DOMContentLoaded会在HTML被解析完后立即执行，而不会被样式表阻塞。

## 总结
DOMContentLoaded和样式表概述
| 浏览器引擎 vs. 行为| 当只有stylesheet link前面有脚本时，样式表是否会延迟DOMContentLoaded | 样式表是否会阻塞随后的外部脚本，因此延迟DOMContentLoaded（测试用例#2） | 样式表是否会阻塞随后内部脚本的执行，因此延迟DOMContentLoaded（测试用例#3）|
| ---- | ---- | ---- | |
| Presto(Opera)   | no    | no   | no |
| Webkit(Safari, Chrome) | no | yes | no |
| Gecko(Firefox)| no | yes| yes |
| Trident(IE)| no | yes| yes |

## HTML5来拯救我们吧

从前，DOMContentLoaded以Mozilla发明的一个Firefox插件。它未经过Javascript社区来审核这是否是一个通用的实现。这就是为什么Opera和Apple使用了这个事件，实现的却不尽相同。因此DOMContentLoaded有着不同的含义。

幸运的是，最重要的web标准:HTML5,将会标准化[DOMContentLoaded][6]事件，和[HTML解析过程][7]

根据HTML5，DOMContentLoaded是一个纯粹的DOM ready事件，与样式表无关。但是，HTML5解析算法要求浏览器延迟脚本的执行，直到前面所有的样式表都加载并解析完毕。让我们复习一下测试用例#2：

```html
<link rel="stylesheet" href="http://molily.de/weblog/stylesheet.css">
<script src="http://molily.de/weblog/script.js"></script>
```

当HTML5解析器遇到`<script></script>`标签时，整个解析进程都被暂停了。首先，当是外部脚本时，浏览器会请求script的资源。然后，浏览器等待前面的样式表加载完毕。第三，Javascript引擎解析并执行下载的Javascript代码。最后，解析器继续解析HTML文档。

假设起码会有一些脚本在样式表后面，它保证了在“Dom ready”的时候所有样式表都加载完毕了。这是因为DOMContentLoaded会在整个文档都解析完毕后触发。

原来Gecko和IE早在这一点上与HTML5达成一致。但是我们不知道Webkit和Opera是否会切换到HTML5解析规范，也不知道触发HTML5解析器是否需要`<!DOCTYPE html>`。

正如上面所说，解析规则是一个巨大的性能影响因素。如果我们把所有样式表和脚本都放在文档的head，那么解析器会等待这些资源全部加载完毕，再去解析body。为了提供更快的性能，基本的原则就是将样式表放在头部，将脚本放在底部。（见[High Performance Web Sites by Stev Souders][8]）

## 原文：
[地址][9]




  [1]: http://javascript.nwbox.com/IEContentLoaded/
  [2]: https://developer.mozilla.org/en/docs/Gecko-Specific_DOM_Events#DOMContentLoaded
  [3]: http://api.jquery.com/ready/
  [4]: http://molily.de/assets/domcontentloaded/chrome-timeline-my.png
  [5]: http://github.com/jquery/jquery/commit/4c1e12e889f2a70bfa3603fed9d1cabe67d294e0
  [6]: http://www.whatwg.org/specs/web-apps/current-work/multipage/the-end.html#the-end
  [7]: http://www.whatwg.org/specs/web-apps/current-work/multipage/parsing.html
  [8]: http://stevesouders.com/hpws/rules.php
  [9]: http://molily.de/domcontentloaded/
  [10]: http://www.whatwg.org/specs/web-apps/current-work/multipage/tokenization.html#parsing-main-incdata
