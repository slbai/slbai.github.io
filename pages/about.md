---
layout: page
title: About
description: Cook the World
keywords: Shi Lin
comments: true
menu: 关于
permalink: /about/
---

I am Shi Lin

A coder who insists on doing something different

Life is short, Why no make it happen

## Concat

{% for website in site.data.social %}
* {{ website.sitename }}：[@{{ website.name }}]({{ website.url }})
{% endfor %}

## Skill Keywords

{% for category in site.data.skills %}
### {{ category.name }}
<div class="btn-inline">
{% for keyword in category.keywords %}
<button class="btn btn-outline" type="button">{{ keyword }}</button>
{% endfor %}
</div>
{% endfor %}
