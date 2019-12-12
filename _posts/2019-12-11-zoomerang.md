---
title: 读zoomerang.js的源码
tags: js库
---

最近利用空闲时间看了两个极简js库(都是来自尤大的Repositoris)的源码实现，有点小收获 🎉

这篇就先来记录一下我对[zoomerang.js](http://yyx990803.github.io/zoomerang/)这个库源码实现的理解：

## zoomerang.js

zoomerang.js是个操作dom的js库，支持放大缩小(几乎)页面中的任何元素

### 整体这个库实现的思路

其实这个库的大体思路很简单，就是通过给dom绑定点击事件去触发放大/缩小（恢复）事件。有意思的是其中的一些细节、动画、以及之前没有见过的原生api。一步步的开始

### IIF

立即执行函数，见了很多js库的源码都是用IIF，zoomerang也不例外，好处在于隔离作用域污染全局变量。以后自己写库的时候也可以借鉴

### 代码格式风格

看了这个库的基本代码结构觉得自己就是个野路子

```
(function() {
  // webkit prefix helper
  ......
  // regex
  ......
  // elements
  ......
  // state
  ......

  // helpers ----------------------------------------------------------------
  function setStyle() {
    ...
  }
  ......
  var api = {
    config: function,
    open: function,
    close: function,
    listen: function
  }

  // umd expose
})()
```

首先把能用到的全局的变量声明，尽量都统一分类写在最前面；
其次就是heplers 功能function 还有 主要的api；
最后做expose；
觉得这是个好习惯，嗯~~~

### css兼容性的处理

这个库的核心就是使用CSS3完成动画，在做CSS兼容性处理的时候 我觉得有几点我可以学的地方

#### WebkitAppearance

```
var prefix = 'WebkitAppearance' in document.documentElement.style ? '-webkit-' : ''
```

用WebkitAppearance属性来检测是否为webkit，倒是没见过这种写法

#### compatibility

```
// compatibility stuff
var trans = sniffTransition(),
    transitionProp = trans.transition,
    transformProp = trans.transform,
    transformCssProp = transformProp.replace(/(.*)Transform/, '-$1-transform'),
    transEndEvent = trans.transEnd
```

定义了一个trans来承载sniffTransition()这个函数处理兼容后的css

```
function sniffTransition () {
    var ret   = {},
        trans = ['webkitTransition', 'transition', 'mozTransition'],
        tform = ['webkitTransform', 'transform', 'mozTransform'],
        end   = {
            'transition'       : 'transitionend',
            'mozTransition'    : 'transitionend',
            'webkitTransition' : 'webkitTransitionEnd'
        }
    trans.some(function (prop) {
        if (overlay.style[prop] !== undefined) {
            ret.transition = prop
            ret.transEnd = end[prop]
            return true
        }
    })
    tform.some(function (prop) {
        if (overlay.style[prop] !== undefined) {
            ret.transform = prop
            return true
        }
    })
    return ret
}
```
这个sniffTransition函数做的就是检查兼容性并且处理兼容的事情，定义了三组css动画兼容的范围（webkit, moz），分别是transiton属性、transform属性、以及transitionend事件（这个transitionend事件是第一次见，可以通过addEventlistener(transitionend)监听到css动画完成）。

然后遍历div.style是否包含兼容性写法的属性，如果包含就添加到ret。这里有一点我当时候比较疑问的为什么要用到some去遍历而不用forEach filter ...，后来一想这个场景用 some 去处理最好不过了，一旦检查到了某个元素符合要求直接就返回true了，同时循环终止不多余浪费性能（警告自己，别一到需要循环遍历的地方就直接无脑的 forEach for）

处理好后的变量transitionProp 、transformProp 、transformCssProp怎么来用，就看到了声明的另一个函数checkTrans

```
function checkTrans (styles) {
    var value
    if (styles.transition) {
        value = styles.transition
        delete styles.transition
        styles[transitionProp] = value
    }
    if (styles.transform) {
        value = styles.transform
        delete styles.transform
        styles[transformProp] = value
    }
}
```
checkTrans 这个功能函数做两件事情，检查传入的style是否存在transition相关的动画属性，如果存在先save属性值，delete这个属性，add处理后的transProp属性，set保留的属性值

### options
```
// options
    var options = {
        transitionDuration: '.4s',
        transitionTimingFunction: 'cubic-bezier(.4,0,0,1)',
        bgColor: '#fff',
        bgOpacity: 1,
        maxWidth: 300,
        maxHeight: 300,
        onOpen: null,
        onClose: null,
        onBeforeClose: null,
        onBeforeOpen: null
    }
```

之前定义的默认的options，包含了动画的时间，过渡的特效，遮盖层的背景颜色，以及放大后的最大宽高，还有四个钩子函数onOpen,onClose,onBeforeClose, onBeforeOpen

### setStyle
```
function setStyle (el, styles, remember) {
    checkTrans(styles)
    var s = el.style,
        original = {}
    for (var key in styles) {
        if (remember) {
            original[key] = s[key] || ''
        }
        s[key] = styles[key]
    }
    return original
}
```

这个函数的作用是先处理兼容性用到了checkTrans函数，然后再覆盖之前的样式。并且根据传入的第三个参数来判断是否需要返回值，如果第三个参数为true 将会返回el的原样式。

### overlay wrapper
创建了两个el元素一个是遮盖层overlay 一个是wrapper

```
setStyle(overlay, {
    position: 'fixed',
    display: 'none',
    zIndex: 99998,
    top: 0,
    left: 0,
    right: 0,
    bottom: 0,
    opacity: 0,
    backgroundColor: options.bgColor,
    cursor: prefix + 'zoom-out',
    transition: 'opacity ' +
        options.transitionDuration + ' ' +
        options.transitionTimingFunction
})
```

通过设置fixed布局 top left right bottom 为0 来撑满整个全屏的效果，dispaly为none。

这里的transition 第三个值是一个cubic-bezier贝塞尔曲线，这个平时几乎用不到但是也了解一下。

wrapper的设置也是fixed布局，left top 都为50%。长宽度都为0的垂直居中的元素。

## api

最主要的核心是api这个对象，包含了 config open close listen四个方法，使用这个库也就是调用api提供的这几个方法就能实现el放大缩小的功能

### api.cofig({...options})

使用这个库之前需要一些配置选项

```
config: function (opts) {

            if (!opts) return options
            for (var key in opts) {
                options[key] = opts[key]
            }
            setStyle(overlay, {
                backgroundColor: options.bgColor,
                transition: 'opacity ' +
                    options.transitionDuration + ' ' +
                    options.transitionTimingFunction
            })
            return this
        }
```

传入一个cofig对象来配置css options, 如果没传入就使用默认的options

### api.open(el, cb)
这个方法就是点击之后触发的事件，放大dom。

两个参数一个是target元素，一个放大事件后的callback，默认是config中提供的onOpen钩子函数。

```
if (shown || lock) return

target = typeof el === 'string'
    ? document.querySelector(el)
    : el

// onBeforeOpen event
if (options.onBeforeOpen) options.onBeforeOpen(target)

shown = true
lock = true
```
之前就定义好的两个变量 show lock 都为false，open放大事件会先判断show\false两个是否为true，有一个为true 就直接return了。先不明白这个地方放在这。

target就是绑定当前点击el，这个地方的写法还巧妙的。 先判断传入的第一个arg是不是String，这样的话 既可以传 '#元素id & .元素class'，又可以直接传这个dom元素。

如果你通过api.config({...options})设置了 onBeforeOpen这个callback的话，那么会在执行放大动画操作前先执行

```
parent = target.parentNode

var p     = target.getBoundingClientRect(),
    scale = Math.min(options.maxWidth / p.width, options.maxHeight / p.height),
    dx    = p.left - (window.innerWidth - p.width) / 2,
    dy    = p.top - (window.innerHeight - p.height) / 2

placeholder = copy(target, p)
```
因为是我自己知识的一个总结，不想像写文章那样说的那么细致 提几个有意思的点

#### dx dy
这两个变量是计算el 自身位置-translate-垂直居中 位置的偏移量，放大动画效果的动画过渡的一段移动距离就是这么来的，这个公式挺好。

#### getComputedStyle(el)
取el的style对象，不需要重复的去写el.style['...']这种代码，可以很轻松复制一个el的样式。

#### el.getBoundingClientRect()
同样返回一个对象，包含left top width height等值。

整个放大的逻辑是这样：

先使用setStyle改变target的样式设置translate(dx, dy)让target偏移，但是会保留原样式赋值给originalStyles这个全局变量。然后往parent父元素里面append遮盖层，和wrapper，另外在target前面添加一个placeholder占位元素（这个占位元素其实就是利用getComputedStyle来复target，设置不可见。）这样做的目的是防止target改变布局会错乱。wrapper是垂直居中的，再往target里添加target。这个时候因为我们先前设置了target偏移，所以el虽在wrapper下，但是却（偏移到）还在原来的位置。

```
overlay.style.opacity = options.bgOpacity
setStyle(target, {
    transition:
        transformCssProp + ' ' +
        options.transitionDuration + ' ' +
        options.transitionTimingFunction,
    transform: 'scale(' + scale + ')'
})
```

这时候改变transform 为scale，并设置transition。 因为覆盖之前的transform属性，现在时刻的target将会不再偏移位置，伴随过渡动画就会回到垂直居中的位置。

这个做法没错，但是我自己尝试去写的时候发现并不是这样，当执行到改变transform为scale的时候，动画似乎并没有移动回垂直位置的过渡轨迹。

#### requestAnimationFrame()

可能是因为浏览器重绘，需要在下次重绘前去更新下一帧动画，我尝试在requestAnimationFrame()中去改变target的transform，之后可以看到完整的动画。
```
requestAnimationFrame(() => {
    target.style['transition'] = `transform .3s cubic-bezier(.4,0,0,1)`
    target.style['transform'] = `scale(2)`
})
```

再看源码里并没有刻意的处理这个事情，也没有用setTime 也没用requestAnimationFrame()。我又看了一下 才发现，在改变target的transform之前声明了一个变量force，但是这个变量也没用。那么我猜想，其实作者声明这个no use的变量就是为了处理这个问题。这个变量通过获取DOM的offsetHeight来主动触发了重绘。这样一想就明白了~~~ (其实是我没学问不认识英文🤭，源码上的备注写的很清楚force layout)

```
// force layout
var force = target.offsetHeight
```

最后在为target添加一个transEndEvent的事件，用来执行onOpen钩子
### api.close(cb)
放大后再点击，触发的缩小恢复事件

传入一个参数，缩小事件后的callback，默认是config中提供的onClose钩子函数

close的逻辑大致和open反着来，将target样式和位置恢复到最初originalStyles，removeChild之前append的overlay,wrapper,placeholder。改变shown 和 el为false。最后执行onOpen钩子。

### api.listen(el)

通过listen来添加click事件, 通过shown 和 el 来判断当前是调用api.close()还是api.open(el)

---

好好学习，天天向上~

<!-- [![Star This Project](https://img.shields.io/github/stars/kitian616/jekyll-TeXt-theme.svg?label=Stars&style=social)](https://github.com/kitian616/jekyll-TeXt-theme/) -->