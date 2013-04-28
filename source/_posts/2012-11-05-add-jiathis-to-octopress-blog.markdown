---
layout: post
title: "向Octopress添加JiaThis分享工具及冲突解决"
date: 2012-11-05 19:08
comments: true
categories: [Octopress]
keywords: Octopress, JiaThis, 冲突, 分享, share, sharing, 框, 小白框, 小框, flash框, flash-video
description: Octopress默认集成twitter、google +1、facebook like等分享工具。添加对JiaThis的支持，JiaThis与Octopress的Video模块有一点小冲突，会导致分享工具条左下角被一个小白框遮盖。解决JiaThis与Octopress冲突。
---

Octopress默认集成twitter、google +1、facebook like等分享工具，只需补全配置文件`_config.yml`中的相关信息即可。

但是，但是，作为水深火热的墙内人民，这些工具大部分状况下都处于瘫痪状态，甚至会拖慢整个站点的加载速度。
权衡起见，只得在`_config.yml`中注释掉twitter、google +1、facebook like相关选项，
并添加对[JiaThis][1]等分享工具的支持。

此外，JiaThis与Octopress的Video模块有一点小冲突，会导致分享工具条左下角被一个小白框遮盖，像一块烦人的膏药，需要做一定处理。

##1. 向_config.yml中添加JiaThis配置开关##

仿照原始风格，在`_config.yml`底部添加如下配置：
```
# JiaThis
jiathis: true

```

以后，想要开启或关闭JiaThis，只需修改该配置值即可。

##2. 集成JiaThis代码##

修改`source/_include/post/sharing.html`，在最后一行`</div>`前添加如下代码：
{% raw %}
``` html
{% if site.jiathis %}
  <!-- JiaThis Button BEGIN -->
  <div class="jiathis_style">
    <span class="jiathis_txt">分享到：</span>
    <a class="jiathis_button_tsina">新浪微博</a>
    <a class="jiathis_button_googleplus">Google+</a>
    <a class="jiathis_button_renren">人人网</a>
    <a class="jiathis_button_qzone">QQ空间</a>
    <a class="jiathis_button_tqq">腾讯微博</a>
    <a href="http://www.jiathis.com/share" class="jiathis jiathis_txt jiathis_separator jtico jtico_jiathis" target="_blank">更多</a>
    <a class="jiathis_counter_style"></a>
  </div>
  <script type="text/javascript" src="http://v2.jiathis.com/code/jia.js?uid=1334720487296344" charset="utf-8"></script>
  <!-- JiaThis Button END -->
{% endif %}
```
{% endraw %}
其中2到14行为从[http://www.jiathis.com/][1]获取的分享代码，可自行定制。

<!-- more -->

##3. 解决JiaThis与Octopress冲突

Octopress会对所有有`movie`属性的`object`标签添加一层形如`<div class="flash-video"><div>`的wrapper，用于视频回放。
实现代码在`source/javascripts/octopress.js`中的`wrapFlashVideos()`函数：
``` js
function wrapFlashVideos() {
  $('object').each(function(object) {
    object = $(object);
    if ( $('param[name=movie]', object).length ) {
      var wrapper = object.before('<div class="flash-video"><div>').previous();
      $(wrapper).children().append(object);
    }
  });
  $('iframe[src*=vimeo],iframe[src*=youtube]').each(function(iframe) {
    iframe = $(iframe);
    var wrapper = iframe.before('<div class="flash-video"><div>').previous();
    $(wrapper).children().append(iframe);
  });
}
```

这层wrapper即是形成JiaThis分享工具条左下角小白框的原因。我们需要对`object`做一个判断，过滤掉JiaThis的object。
将`wrapFlashVideos()`函数改为以下代码即可：
``` js
function wrapFlashVideos() {
  $('object').each(function(object) {
    object = $(object);
    if (object.attr('id') != "JIATHISSWF") {
      if ( $('param[name=movie]', object).length ) {
	var wrapper = object.before('<div class="flash-video"><div>').previous();
	$(wrapper).children().append(object);
      }
    }
  });
  $('iframe[src*=vimeo],iframe[src*=youtube]').each(function(iframe) {
    iframe = $(iframe);
    var wrapper = iframe.before('<div class="flash-video"><div>').previous();
    $(wrapper).children().append(iframe);
  });
}
```

[1]: http://www.jiathis.com/
