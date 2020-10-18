---
layout: page
title: About
description: 搞代码的
keywords: 搜索引擎 读书 思维
comments: true
menu: 关于
permalink: /about/
---

我是吴鹏，一枚字节跳动的小螺丝钉。

我最近制定了一份为期33年的计划，去100座城市爬100座山。



   
## 联系

<ul>
{% for website in site.data.social %}
<li>{{website.sitename }}：<a href="{{ website.url }}" target="_blank">@{{ website.name }}</a></li>
{% endfor %}

<li>
微信：<br />
<img style="height:192px;width:192px;border:1px solid lightgrey;" src="{{ assets_base_url }}/assets/images/wechat.png" alt="兜里有块糖" />
</li>

</ul>


## Skill Keywords

{% for skill in site.data.skills %}
### {{ skill.name }}
<div class="btn-inline">
{% for keyword in skill.keywords %}
<button class="btn btn-outline" type="button">{{ keyword }}</button>
{% endfor %}
</div>
{% endfor %}

---

>欢迎加入字节跳动，悄摸摸的内推入口:
>
>社招：https://job.toutiao.com/s/J5JPnyA
>
>校招：字节跳动校招内推码: 1XBE4ET
   投递链接: https://job.toutiao.com/s/J5JMmGM