---
layout: page
title: Daily english
description: Daily English提供每日英语学习资源，包括单词、短语、句子和文章，帮助您提高英语水平。
keywords: 每日英语学习、英语单词、英语短语、英语句子、英语文章
comments: false
mermaid: false
menu: Daily
permalink: /daily/
---

> When the sun came up… I couldn’t tell where heaven stopped and the Earth began.

{% assign daily = '' %}
{% for item in site.daily %}
{% if item.title != "Daily Template" %}
  {% for tag in item.tags %}
    {% if daily contains tag %}
    {% else %}
      {% if daily != '' %}{% assign daily = daily | append: ',' %}{% endif %}
      {% assign daily = daily | append: tag %}
    {% endif %}
  {% endfor %}
{% endif %}
{% endfor %}

{% assign taglist = daily | split: ',' | sort_natural %}

<a href="{{ site.url }}/daily/" style="color:#888;display:inline-block;margin:0 8px;">All</a>{% for tag in taglist %}<a href="{{ site.url }}/daily/?tag={{ tag }}" style="color:#888;display:inline-block;margin:0 8px;">{{ tag }}</a>{% endfor %}

<ul class="listing">
{% for item in site.daily %}
{% if item.title != "Daily Template" %}
<li class="listing-item" tags="{% for tag in item.tags %}{{ tag }} {% endfor %}">
  <a href="{{ site.url }}{{ item.url }}">{{ item.title }}</a>
  {% for tag in item.tags %}
  <a style="font-size:12px;color:gray;font-style:italic;display:inline-block;margin:0 0 0 4px;padding:0 4px;background-color:lightgray;" href="{{ site.url }}/daily/?tag={{ tag }}" title="{{ tag }}">{{ tag }}</a>
  {% endfor %}
</li>
{% endif %}
{% endfor %}
</ul>

<script>
jQuery(function() {
    function getUrlParam(name) {
        var reg = new RegExp("(^|&)" + name + "=([^&]*)(&|$)");
        var r = window.location.search.substr(1).match(reg);
        if (r != null) return r[2]; return null;
    }

    var tag = getUrlParam('tag');
    if (tag == undefined || tag === '') {
        return;
    }

    $(".listing-item").each(function() {
        if ($(this).attr('tags').indexOf(tag) < 0) {
            $(this).css('display', 'none');
        }
    });

});
</script>
