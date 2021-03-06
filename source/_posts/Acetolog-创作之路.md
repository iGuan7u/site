---
title: Acetolog 创作之路
date: 2020-02-21 17:17:25
tags:
---
## 前言
看到陆续有些人使用自己的博客主题 [Acetolog](https://github.com/iGuan7u/Acetolog)，感觉也是时候注水一下关于这个博客主题了。

<!-- more -->

## 起因
其实跟很多技术同学的出发点相同，就是想拥有一个自己的技术博客，有机会的话能够将自己的专属 ID **iGuan7u** 发扬光大，然后陆续开始筹备整个方案，购置 VPS、购买域名、选择博客系统、挑选博客主题…然后也跟许多同学的发展路线一致：购置准备相关的东西都是一帆风顺的，直到**挑选博客主题**这一步遇到了瓶颈。

能够遇到符合自己审美的主题实在太难了，笔者也是寻遍了整个 Google，才偶然遇到了这个主题 [typology](https://demo.mekshq.com/typology/)，甚至后来为了这个主题不得已地从其他的博客系统毅然决然地投奔到 wordpress 的怀抱中。~~因为它实在是太美了。~~

当然，后面的故事也能想象到：wordpress 对于一个简单的博客系统来说实在是太大了，有很多不必要的功能、同时也因为过于庞大，导致想实现自己的功能都无从下手。另外 typology 虽然非常美观，可是页面加载速度却不甚理想，其中包含了很多自己并不想要的功能跟表现。

## 动手
当然，开始启动的时候，还是希望像素级的还原（抄袭）typology 的主题的，由于自己的懒惰，以及目前本人博客的文章数量还不多，后续对 typology 的功能进行了精简，文章详情部分样式还做了自己的发挥。~~虽然现在看来甚至还不如不发挥。~~

不过样式部分其实并不属于这个博客的精彩部分。

笔者在创作这个主题是其实最关心的部分，是博客所需加载的资源大小。笔者对比过 typology，对比过 hexo 主题界几乎一统江山的 [hexo-next](https://github.com/theme-next/hexo-theme-next) ，acetolog 所需加载的资源数量以及资源大小都是有极大优势的。

![hexo-next](https://cdn.iguan7u.cn/image/hexo-next_resources.png)

![typelogy](https://cdn.iguan7u.cn/image/typology_resources.png)

![acetolog](https://cdn.iguan7u.cn/image/acelog_resources.png)

同时，不同于广泛的博客主题，在 Acetolog 中，Javascript 在其中所发挥的，仅仅是提升用户体验的作用，Acetolog 甚至能在浏览器禁用 Javascript 运行的环境下正常打开。

虽然这部分优势对于普通的读者来说是毫无作用的，可是对于笔者来说，这取得的成就感可以说是前所未有的。毕竟，搭建博客的过程本身就是一件值得享受的事情。感兴趣的读者可以 fork 一下，跟笔者一起完善这个主题。

## 感想
另外一点关于 Javascript，自从 Google 使用 Javascript ajax 做搜索候选关键字后，Javascript 这门语言可以说得到了飞速的发展，现在的 React、Vue.js 的框架更是将 Javascript 推上了不可撼动的地位，以至于如今的前端页面，没有 Javascript，甚至根本无法正常运行。（有兴趣的读者可以尝试自行在浏览器中关于 Enable Javascript 的选项，看看那个网页还能正常展示）这里并非说 Javascript 不好，只是在目前，这种当初仅仅作为浏览器与网页间沟通的语言，过多地承载了负责展示页面信息的功能，使得前端工作似乎仅仅是把 Javascript 写好而已，其中的 HTML 跟 CSS 就黯然失色。

自己在下定决心实现这个主题前，自己也有寻找过很多主题，可是实在看不惯很多开发者，在为了简化 DOM 选择、或者是方便 CSS 属性变更，就直接把 `jQuery` 这个库引入博客资源，这对整个博客的加载速度造成了多大的影响。（扶额）

甚至，当初 Acetolog 根本不打算引入 Javascript，包括右上角的 Sidebar 功能，笔者都找到了单纯使用的 HTML5 + CSS 解决方案，但是后面发现自己在这个方面有点过于执着了，也极大的限制了这个博客主题的发展，很庆幸自己很快便认清了现实。

目前阶段 Acetolog 还是以 **tiny** 为主要目的，为大家提供一个精简而又优雅的博客主题。对于功能上的开发会尽可能地克制，至少自己能保证，每一行新增的代码，我都考虑了加载速度的影响。

谢谢。