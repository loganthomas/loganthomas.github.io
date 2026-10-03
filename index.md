---
layout: page
---

<img src="/assets/images/profile_pic.jpg" alt="Logan Thomas profile picture" class="profile-image" loading="lazy">
<h1 class="text-center">Logan Thomas</h1>
<br/>
I'm a data science software engineer at [Fullstory](https://www.fullstory.com/){:target="_blank"}, passionate about building software and sharing knowledge. This site is where I share what I'm building, blog about open source contributions and other software musings, and keep a record of the teaching and talks I've given.

[Learn more about me →](/about/)

<hr class="section-divider">

{% include working_on.html %}

<hr class="section-divider">

<h3>Recent Posts</h3>
{%- if site.posts.size > 0 -%}
<ul class="post-list">
    {%- for post in site.posts limit:5-%}
    <li>
    {%- assign date_format = site.minima.date_format | default: "%b %-d, %Y" -%}
    <span class="post-meta">{{ post.date | date: date_format }}</span>
        <a class="post-link post-link-large" href="{{ post.url | relative_url }}">
        {{ post.title | escape }}
        </a>
    {%- if site.show_excerpts -%}
        {{ post.excerpt }}
    {%- endif -%}
    </li>
    {%- endfor -%}
</ul>
{%- endif -%}

<hr class="section-divider">

<h3>Recent Talks</h3>
<ul class="post-list">
    {%- assign shown = 0 -%}
    {%- for post in site.posts -%}
    {%- if shown == 3 -%}{%- break -%}{%- endif -%}
    {%- if post.type == "talk" or post.type == "tutorial" -%}
    {%- assign shown = shown | plus: 1 -%}
    <li>
    {%- assign date_format = site.minima.date_format | default: "%b %-d, %Y" -%}
    <span class="post-meta">{{ post.date | date: date_format }}</span>
        <a class="post-link post-link-large" href="{{ post.url | relative_url }}">
        {{ post.title | escape }}
        </a>
    </li>
    {%- endif -%}
    {%- endfor -%}
</ul>

[See all Teaching & Talks →](/teaching/)
