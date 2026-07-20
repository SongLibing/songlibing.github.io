---
layout: page
title: English Blogs
icon: fas fa-language
order: 1
permalink: /english/
---

Articles written in English. You can also subscribe via the [English feed](/feed-en.xml){: target="_blank" }.

{% assign en_posts = site.posts | where_exp: "post", "post.lang == 'en'" %}

{% if en_posts.size == 0 %}
_No English articles yet._
{% else %}
<ul class="content">
{% for post in en_posts %}
  <li style="margin-bottom: 1.2rem;">
    <a href="{{ post.url | relative_url }}" style="font-size: 1.1rem; font-weight: 600;">{{ post.title }}</a>
    <div style="color: var(--text-muted-color); font-size: 0.85rem; margin: 0.2rem 0;">
      {{ post.date | date: "%Y-%m-%d" }}
    </div>
    <div style="color: var(--text-muted-color);">
      {{ post.content | strip_html | truncatewords: 40 }}
    </div>
  </li>
{% endfor %}
</ul>
{% endif %}
