---
layout: page
title: Chinese Blogs
icon: fas fa-book
order: 2
permalink: /chinese/
---

中文文章。也可以订阅[中文 feed](/feed-zh.xml){: target="_blank" }。

{% assign zh_posts = site.posts | where_exp: "post", "post.lang != 'en'" %}

{% if zh_posts.size == 0 %}
_暂无中文文章。_
{% else %}
<ul class="content">
{% for post in zh_posts %}
  <li style="margin-bottom: 1.2rem;">
    <a href="{{ post.url | relative_url }}" style="font-size: 1.1rem; font-weight: 600;">{{ post.title }}</a>
    <div style="color: var(--text-muted-color); font-size: 0.85rem; margin: 0.2rem 0;">
      {{ post.date | date: "%Y-%m-%d" }}
    </div>
    <div style="color: var(--text-muted-color);">
      {%- assign _sum = post.description | default: site.data.summaries[post.slug] -%}
      {%- if _sum -%}
        {{ _sum }}
      {%- else -%}
        {%- assign _body = post.content -%}
        {%- assign _parts = post.content | split: '</blockquote>' -%}
        {%- if _parts.size > 1 -%}{%- assign _body = _parts[1] -%}{%- endif -%}
        {{ _body | strip_html | strip_newlines | truncate: 90 }}
      {%- endif -%}
    </div>
  </li>
{% endfor %}
</ul>
{% endif %}
