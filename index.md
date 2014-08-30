---
layout: page
title: Home Page
tagline: welcome 欢迎
---
{% include JB/setup %}

{% assign preview_limit=5 %}
{% assign preview_words=100 %}

<a href="feed.xml">
<img src="{{ ASSET_PATH }}hooligan/images/rss.gif" width="36" height="14">
</a>

<div class="blog-index">  
{% for post in site.posts limit:preview_limit %}
    <h2 class="entry-title">
        {% if post.title %}
            <a href="{{ root_url }}{{ post.url }}">{{ post.title }} </a>
        {% endif %}
    </h2>
    <div class="entry-content">
        {% if post.content contains '<!--more-->' %}
            {{ post.content | split:'<!--more-->' | first }}
        {% else %}
            {{ post.excerpt }}
        {% endif %}
    </div>
    {% if post.title %}
        <a href="{{ root_url }}{{ post.url }}">read more</a>
    {% endif %}

<hr />

{% endfor %}
</div>


{% if site.posts.size > preview_limit %}
<h2> Other posts <h2>
<ul class="posts">
  {% for post in site.posts offset:preview_limit %}
    <li><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>
{% endif %}
