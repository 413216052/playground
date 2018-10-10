# `一次要了老命的分享React-Virtual DOM`
@(React)[DOM - Virtual DOM|diff|Fiber]

[TOC]

-------

## 前言
有的时候我们随手写一段代码，在浏览器中跑起来发现毫无问题，但是如果放在“第一次前端革命”之前，就没这么简单了，那这是为什么呢？

`“不就是一个DOM吗，有什么了不起”`

所谓站在巨人的肩膀上写代码，就是说哪怕是一个DOM操作，也经过了许许多多的革新，我们拿当下项目（通天塔-可视化）用的框架React来讲解这点。
react作为某种程度上的主流前端框架，它有着许许多多的优点和思想，这些使得前端er们构建项目变得得心应手。不过仔细想一想，浏览器作为一个引擎，提供了很多出色的功能，在一个重型应用中，项目的代码很复杂，参与的码农们也不少，然而能够流畅的跑起来，这是不是也有点太过顺畅了？

那么是什么给我们提供了这种流畅的撸码体验，在肆无忌惮的撸码后，React帮助我们实现了什么呢？

-------------------

React中，render执行的结果得到的不是真正的DOM节点，其结果是相比DOM较为轻量的js对象，即Virtual DOM。

相比于真实DOM，Virtual DOM有什么不同？
在此之前我们先深入了解下DOM。

------------------

## 关于DOM
从MDN上找到的官方解释：

> `[ 0 ]` Document Object Model（文档对象模型）是HTML和XML文档的编程接口。它提供了对文档的结构化的表述，并定义一种方式可以使从程序中对该结构进行访问，从而改变文档的结构、样式和内容。DOM将文档解析为一个由节点和对象（包含属性和方法的对象）组成的结构集合。

可以理解为：**DOM是一套对文档的内容进行抽象和概念化的方法**

DOM本身是一种模型，表现了页面文档的结构，所以在各种语言下，DOM都有它最基础的属性和方法，可以理解为它们合在一起而构成的数据实体。

DOM把一份文档表示为一份树形结构，一般用parent（父）、child（子）和sibling（兄弟）来表明成员之间的关系。那么这一份数据结构，称之为DOM树，而他的基本单元就是DOM节点。

balabalabala...
[此处省略万把字][1]

老生常谈的问题：都说DOM操作损耗性能，那么为什么会损耗性能，一个DOM节点里都有什么，操作这些又会发生什么？

->> 在控制台创建一个空节点：
``` javascript
var qwe = document.createElement('p')
```
->> console出来
``` javascript
console.dir(qwe)
```
可以看到 qwe 这个div节点本身具有着很多很多属性，其中有我们熟知的：
- `childNodes` - []
- `className` - String
-  `offsetHeight` - Number
-  `onblur` - null
-  `onclick` - null
-  `oncopy` - null
-  `onkeydown` - null
-  `onload` - null
-  `onmousemove` - null
-  `onscroll` - null
-  `innerHTML` - null

除此之外，还可以看到这个空节点继承的类（原型链）：
`HTMLDivElement` ->> `HTMLElement` ->> `Element` ->> `Node` ->> `EventTarget` ->> `Object`

点开每一层原型链，都可以看到其中不仅仅有着基础的数据属性，同时包含了很多访问器属性，仔细观察不难发现，构建一个div标签、a标签或者input标签具有着较为不同的属性和原型链，即：
> `[ 1 ]`每个DOM节点属于相应的内置类

附一张图可以明显的认识到其中的关系

![dom-node-properties](https://www.w3cplus.com/sites/default/files/blogs/2018/1805/dom-node-properties-1.png)
<br>
对比其他js对象，我们列一个比较：
- 超轻量 Object.create(nulll)
- 轻量 一般的对象 {}
- 重量 带有访问器属性的对象,（getter、setter）
- 超重量 各种节点或window对象

`“原来在js中，DOM对象有这么深的水，难怪DOM操作这么慢”`

到现在为止我们看到的都是js对象的属性及其原型链，进行的操作也只有创建DOM，负责任的说，DOM操作并不是`特别慢`，造成“DOM操作损耗性能”这一说法，其实另有原因。
先说明DOM操作“不慢”的原因，由于时间不充裕，我没有做实际评测，就上一张前人的实测图以做参考：
![dom-dataset-innerHTML](https://pic3.zhimg.com/80/v2-eff1d99cf8b44214d05bd923ceee1ede_hd.png)
<br>
稍微会比访问 dataset 慢一些，但是比访问 innerHTML 要快得多
> [ 2 ]从 javascript 中访问 DOM 其实确实会导致一定的性能损耗，因为 DOM 本质上不属于 js 的内容，而是一组桥接 document 和 js 世界的桥梁（API）。由于js对像的操作很快，所以对 DOM 这种 API 访问的损耗其实是可以忽略不计的，哪怕它嵌套n层继承n层。真正慢的是因为不能足够了解到 `浏览器渲染机制`而不恰当的访问 DOM ，会造成浏览器的「重排」，这才是性能的杀手！所以从上图可以看到，对 innerHTML 访问的性能是最差的。

那么为什么重排会损耗大量的性能呢？
先复习一下浏览器渲染流程：
- 解析HTML，并生成一棵DOM tree
- 解析CSS，生成CSSOM（CSS Object Model）树
- 加载并执行JavaScript代码（內联代码和外联代码）
- 解析各种样式（css）并结合DOM tree生成一棵Render tree（按顺序展示在屏幕上的一系列矩形，这些矩形带有字体，颜色和尺寸等视觉属性）
- layout：对Render tree的各个节点计算布局信息
- painting：根据Render tree并利用浏览器的UI层进行绘制

可以看到浏览器渲染流程中，从DOM tree的生成到绘制出视图，其中不乏有各种js、css阻塞，在有考虑过足够的性能优化下，排除一些非确定因素，浏览器还是需要做大量的几何计算。

Webkit渲染引擎流程
![webkit](https://pic1.zhimg.com/80/v2-219c89392774bcc81bac826f8e51cccd_hd.jpg)
<br>
Gecko渲染引擎流程
![webkit](https://pic1.zhimg.com/80/v2-219c89392774bcc81bac826f8e51cccd_hd.jpg)
<br>

拿一个单页应用项目来讲，首屏加载成功后，用户在页面上进行的DOM操作，百分之九十九都会引起上图中的layout步骤，即重排（reflow），此时，所有被影响的节点都会重新计算，计算好后再次绘制。

`“我懂了，如果影响的DOM节点越多，reflow+repaint次数就越多”`

浏览器开发大大们深刻认识到DOM操作对视图带来的危害，所以现代浏览器中，已经对次进行了优化：一般情况下，浏览器的layout是lazy的，即：js执行过程中，Render tree 不会及时更新的，所有对DOM的修改都会存在一个队列中，然后在最后进行layout。
所以说，重排也不能说是损耗性能的罪魁祸首。

再次试想一个场景（嗯...时间有限就不做实战演示了），在某段js代码中，需要提取准确的节点位置信息，那么如果没有被绘制出来的话，怎么才能拿到准确的信息呢？
没办法，这时候就只能强制layout，此时浏览器本身的lazy优化机制便失去了作用。

假设这么一段代码：
```javascript
div.style = {
	display: 'none',
	transition: 'all .3s ease',
	opacity: '0',
	width: '100%',
	height: '100px'
} 
	// button-function.....
div.style = {
	display: 'block',
	opacity: '1'
}
```
预想的div在某种操作后（button）由透明变为实体，但是一般情况下，透明度的渐变并不能生效，是因为displai属性和opacity属性被堆在浏览器渲染队列里，那么display在none->block的同时，想要opacity渐变是不可能的，所以想要达到效果，就要这么改一下：
```javascript
...
// button-function.....
div.style.display = 'block'
console.log(div.offsetWidth)
div.style.opacity = : '1'
```
其中的原理就是，HTMLElement.offsetWidth 是一个只读属性，返回一个元素的布局宽度，访问这个属性就会强制浏览器进行layout，统计了能强制layout的常用操作：
- 计算此渲染对象的布局信息（HTMLElement.offsetLeft, HTMLElement.offsetTop, HTMLElement.offsetWidth, HTMLElement.offsetHeight, HTMLElement.offsetParent）
- 变更内容（HTMLElement.innerText）
- 激活伪类（HTMLElement.focus()）
- 访问或改变某些CSS属性（top、height）
- 浏览器窗口变化（window.scrollX, window.scrollY）

这么算下来实际上强制重排的情况还是蛮多的，尤其在现在各类web-app层出不穷、实现各类你想不到的业务场景下，操作DOM也是必须的了。

回到上文提到的DOM并不慢，比方说绑定事件，并不会造成浏览器重排，因为仅仅只是js对象的操作。

由于操作DOM具有较大的代价，并且由于我们大部分的代码都不会去为每次操作做性能优化，如果一次状态（state）的改变引起了大规模DOM操作，那么浏览器可能会不断的进行重排，这部分的性能花销非常大，于是后来就有了Virtual DOM的出现。

------------------

## 关于React Element
Virtual DOM 是真实 DOM 的模拟，而React为了应对频繁的 DOM 操作，也使用了Virtual DOM，即React Element类，不过与vue这种在数据层面检测的方式不同，React的检查是 DOM 结构层面的。

**什么是React Element**
先拿通天塔-可视化项目中的组件来说，这样比较直观：
```javascript
//点击「金融券」
render() {
    const props = this.props
    let tabs =
    (<Tabs
        ref="i am a ref property"
        key="key is key"
        className="coupon-prop prop-pane__tabs finance_coupon"
        onBeforeChange={this.props.onBeforeTabChange}>
        <TabPane name={TAB.STYLE}>
            <PropPane
                {...props}
                sections_i={this.baseSections_i}
                onValidate={this._validate}
                extendFields={this.extendFields}
                onChange={this._onPropsChange}
            />
        </TabPane>
        <TabPane name={TAB.DATA} mode="always">
            <DataPane {...props} getDefaultCoupon={getDefaultCoupon} />
        </TabPane>
    </Tabs>)
    console.clear()
    console.log(tabs)

    return tabs
}
```
<br>
直接打印出这个“节点”，看会出现什么：
```javascript
{
	$$typeof: Symbol(react.element),
	key: "key is key",
	props: {className: "coupon-prop prop-pane__tabs finance_coupon", onBeforeChange: ƒ, children: Array(2), activeKey: 0},
	ref: "i am a ref property",
	type: ƒ Tabs(props),
	_owner: Ko {tag: 2, key: "viewInstance815", type: ƒ, stateNode: ProxyComponent, return: Ko, …},
	_store: {validated: false},
	_self: null,
	_source: null,
	__proto__: Object
}
```
<br>
多打印几个组件，你会发现它们大同小异，结构基本一样，其实这就是React Element，当然打印组件不能明显感觉到它的特异性，我通过调用源码来给你展示一个button以作比较：
```javascript
 let config = {
     className: 'button',
     type: 'button'
 }
 console.log(React.createElement('button', config, 'Click'))
```
```javascript
{
	$$typeof: Symbol(react.element),
	key: null,
	props: {className: "button", type: "button", children: "Click"},
	ref: null,
	type: "button",
	_owner: Ko {tag: 2, key: "viewInstance815", type: ƒ, stateNode: ProxyComponent, return: Ko, …},
	_store: {validated: false},
	_self: null,
	_source: null,
	__proto__: Object
}
```
<br>
其中有部分属性能直观的理解，有一些需要查资料，整理好先抛在这：

- [ ] key // 没错就是那个key

- [ ] ref // 给节点打标，如果想获取改节点的布局信息，通过这个ref来访问，为DOM操作作基础\

- [ ] type // String or Function

- [ ] _ _ proto _ _ // 直接指向Object

- [ ] $$typeof 这个属性有点特别，怎么说呢，看上去它最贵重，而且都是Symbol(react.element)，查了下资料：
> All React elements require an additional 美元美元typeof: Symbol.for('react.element') field declared on the object for security reasons. It is omitted in the examples above. This blog entry uses inline objects for elements to give you an idea of what’s happening underneath but the code won’t run as is unless you either add 美元美元typeof to the elements, or change the code to use React.createElement() or JSX.

洋文翻译过来，大致就是讲 美元美元typoef会在jsx中自动添加进来，用来解决安全问题，至于想直到什么安全性问题的，事后你们自己看以下链接吧
- https://github.com/facebook/react/pull/4832
- https://github.com/facebook/react/issues/3473
- http://danlec.com/blog/xss-via-a-spoofed-react-element
- https://hackerone.com/reports/49652
<br>
- [ ] props 嗯，就是传入的props，打印 tabs 组件这个属性可以看到：
```javascript
// console.log(tabs.props)
{
	activeKey: 0,
	children: Array(2)
		0: {$$typeof: Symbol(react.element), props: {…}, …},
		1: {$$typeof: Symbol(react.element), props: {…}, …},
	length: 2,
	__proto__: Array(0),
	className: "coupon-prop prop-pane__tabs finance_coupon",
	onBeforeChange: ƒ (),
	key: undefined,
	ref: undefined,
	get key: ƒ (),
	get ref: ƒ (),
	__proto__: Object
}
```
<br>
可以看到，children属性里有着和当前对象相同的...

`"哇，我知道了，这tm就是一个tree结构！"`

至于这个对象里别的属性，都是这个我们项目里的tabs组件里写好的东西，就不一一说了。

- [ ] _owner 负责创建这个 React 元素的组件，不过在 0.13 就被废弃掉了，所以不管它了
<br>
- [ ] 其他属性只供开发环境下用的，时间有限，就不一一解释了

不过到这相信你也大致明白了，React是怎么表示它的 Virtual DOM 节点的。

------------------

##React Element 都干了些什么
`“React Element我理解了，那么它都干了些什么呢？”`

上文说到过DOM的各种操作，由于浏览器已经做了渲染优化，但是部分操作还是会强制浏览器layout，造成大量的性能损失，此外，真实的DOM变动也会引起别的事情发生，你可以想象一个节点的各种监听事件，在一个庞大的单页应用中，发生了变化时，浏览器会进行怎么样计算，可以肯定的说其损耗的能源是巨大的，感兴趣可以去看一下能源损耗图，我上张我的（题外话）：
![nengyuan](https://m.360buyimg.com/babel/jfs/t27394/289/930822648/283724/500b4a40/5bbccb86N04b2ee9d.jpg)
当然这不能解释什么，只不过旁敲侧击的告诉你现在开发的项目是有多“重”，如果我们不用现代框架，开发出来的东西也许比现在还要重的多。

那么Virtual DOM就可以解决这个问题，怎么说呢，你大致可以这么理解，用js对象的计算来代替浏览器布局计算，直到一个`事务`的尽头，即改执行的都执行完了，在最后需要刷新视图的时候进行一次真实的Render tree渲染。

我们取一个模板的配置面板作参考，就拿「金融券」那个`tabs`来举栗吧：
![tabs](https://m.360buyimg.com/babel/jfs/t27733/246/892049003/288947/f2ea6cc1/5bbccf05N1ea4e188.jpg)

点开“样式配置”：
![tabs-style](https://m.360buyimg.com/babel/jfs/t26503/359/906595573/320909/28987821/5bbcd0c2N7ceeb078.jpg)

可以看到就这么一个模板配置项中的tabs就嵌套了很多组件，然后进行一次配置项改动（部分配置项改动可以引起其他配置项节点增删），先找到对应的源代码：
![onchange](https://m.360buyimg.com/babel/jfs/t25012/187/1827665960/147782/4220aaed/5bbcd283N5385f479.jpg)

更改「背景图-背景色」作出性能检测：
![performance](https://m.360buyimg.com/babel/jfs/t24868/14/1862175422/270797/eece3dac/5bbcd667Nd1948214.jpg)

这时已经能看出来，浏览器计算的大头已经不是渲染了，而是Scripting。

`"那Scripting中都做了些什么事情呢？"`

很多事情，从鼠标点击、事件捕获、目标阶段、冒泡阶段、react生命周期，再到我们的onChange，这些我们统统都不讲，我就拿 Virtual DOM 改变后说事。

上文说到了虚拟节点对象的结构，随便领出来一个节点（只要不是最末尾那个）就是一个节点树，所以数形结合一下，可以大致画出简易的树形结构图：

![compare](https://m.360buyimg.com/babel/jfs/t27331/297/972632452/305427/f3164515/5bbdd165N67807496.jpg)
<br>
左面的是改变前，右边的改变后，先理一下思路，这两个都是不做视图更新的，他只是react作了改变前后的Virtual DOM树比较，可以看到变颜色的框中，只有蓝色的框中一个节点改变，而react就是在蓝色框框中比较出了d1 - d2 的改变；

而至于它是怎么比较的，这涉及一个react核心算法 — React diff，不过在此之前，我可以先简单的了解下`传统diff算法`本身的思路。

算了没有中文文档，只找到英文的，实在是看不懂，感兴趣自己去看吧

--> [传统diff讲解][11]

我算是知道为什么所有的博客和文章都对传统diff一笔带过了，我直接上结论：
传统diff通过循环递归对节点进行依次对比，为了达到完全的对比，并作出最小的修改，算法复杂度达到 O(n^3)，如果一个项目有1000个节点，那么diff一下就是 10^9 次比较，嗯，会爆炸。

可是本就是为了性能，这样做肯定是不划算的，所以 react 要制定新的策略放弃了`完全比较`和`最小改动`，将算法复杂度做到 O(n)。
- DOM 节点跨层级的移动操作不属于成熟的编码规范，所以情况很少（tree diff）
- 拥有相同类的两个组件将会生成相似的树形结构，拥有不同类的两个组件将会生成不同的树形结构（component diff）
- 同一层级的子节点，通过key打标记录（element diff）

**Tree diff**：可以理解为分层diff，因为不考虑跨层级移动节点，所以仅在同一层级下对比节点。
![tree-diff](https://pic2.zhimg.com/80/v2-8ea9da2b9d53796c8902b5994158c8f9_hd.jpg)
<br>
**Component diff**：比较组件的Type，如果type一致，继续比较virtual DOM tree，否则替换全部当前节点和它的子节点
![component-diff](https://pic1.zhimg.com/80/52654992aba15fc90e2dac8b2387d0c4_hd.png)
<br>
**Element diff**：当节点处于同一层级时，有三种节点操作：插入、移动、删除
- 插入：新节点树的 component 类型不在老节点树里，那就是全新的节点，直接插入
- 移动：老节点树有新的 component 类型，对比新老节点树发现有相同的element节点，只不过位置变了，于是进行移动
- 删除：新节点树没有老节点树的 component类型；或者两方都有，但element节点需要被删除，这两种情况进行删除操作

![element-diff](https://pic1.zhimg.com/80/7b9beae0cf0a5bc8c2e82d00c43d1c90_hd.png)
<br>
到目前为止，事实证明三个策略普适且合理，嗯，对大佬们高山仰止。

回到我们自己项目中来，我们点击切换「背景色」-> 「背景图」，代码一顿运行之后，发现 Virtual DOM tree 中的 d1 节点被更换，到这时这个diff已经对比出来，那么之后就要将改动告诉真实DOM tree，这个步骤称为`patch`，让真实的DOM tree为之改变，改变一个真实的d1 -> d2，之后再交给浏览器去布局渲染，这么一整个过程，我们用一个官方的词来描述：调和。
有了调和，就规避了原始DOM操作带来的性能损耗了，再扒一张图来概括：
![diff-patch](https://user-gold-cdn.xitu.io/2018/4/18/162d461714ff4e2d?imageslim)
<br>

嗯，时间有限，diff就说这么多。

------------------
`"你说的我都明白了，但是，Virtual DOM是怎么映射成真实DOM的呢？"`

给你一个链接自己看吧，这次是真的没时间了！
[从react源码看Virtual Dom到真实Dom的渲染过程][9]

----------------------
## Fiber

`"如果React Element很强大，为什么我在写节点动画的时候也会经常遇到“卡顿”的现象呢？"`

其实React 在每次收到数据更新之后，会进行一次调和过程并更新 DOM，一般来讲都给给玩家们提供不错的性能，但在一些较为变态的场景下，比如动画、手势，连续的DOM更新下，视图会来不及响应，在v16版本前，这一直是react的痛点。而在v16引入了一个新的概念 — Fiber（纤程），为新一代的调和过程提供基础，它的主要特性为：支持增量渲染，把渲染工作切片。
之所以这样做，是因为在js-eventloop中是会一直占用浏览器主线程的，这个不结束，渲染就不执行，至于它是怎么做的，说实话我也不清楚，如果大家有兴趣学习，可以一起交流。

送给你一个英文版的学习视频，如果你能将它翻译过来，相信出版费就能让你吃喝不愁
https://www.youtube.com/watch?v=ZCuYPiUIONs

另外钩咸饵直的放个图，Fiber中用到的堪称神器的API — requestIdleCallback

![requestId](https://m.360buyimg.com/babel/jfs/t27712/190/974069181/69399/2d099d6c/5bbdf99aN1b588d03.jpg)
<br>
这是在我们的项目中console出来的～

-----------
> References：
> https://www.w3cplus.com/javascript/intro-dom.html
> https://www.w3cplus.com/javascript/node-properties-type-tag-and-contents.html
> https://www.zhihu.com/question/54966411
> https://segmentfault.com/a/1190000004114594
> https://zhuanlan.zhihu.com/p/26105913
> https://www.zhihu.com/question/31809713/answer/80089685
> https://www.w3cplus.com/javascript/understand-the-Virtual-DOM.html
> https://github.com/creeperyang/blog/issues/30
> https://www.jianshu.com/p/df0b5a009e92
> https://zhuanlan.zhihu.com/p/36500459
> https://grfia.dlsi.ua.es/ml/algorithms/references/editsurvey_bille.pdf
> https://zhuanlan.zhihu.com/p/20346379

  [1]: https://www.w3cplus.com/javascript/intro-dom.html
  [2]: https://www.w3cplus.com/javascript/node-properties-type-tag-and-contents.html
  [3]: https://www.zhihu.com/question/54966411
  [4]: https://segmentfault.com/a/1190000004114594
  [5]: https://zhuanlan.zhihu.com/p/26105913
  [6]: https://www.zhihu.com/question/31809713/answer/80089685
  [7]: https://www.w3cplus.com/javascript/understand-the-Virtual-DOM.html
  [8]: https://github.com/creeperyang/blog/issues/30
  [9]: https://www.jianshu.com/p/df0b5a009e92
  [10]: https://zhuanlan.zhihu.com/p/36500459
  [11]: https://grfia.dlsi.ua.es/ml/algorithms/references/editsurvey_bille.pdf
  [12]: https://zhuanlan.zhihu.com/p/20346379