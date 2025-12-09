---
layout: page
title: Turkey Bowl Archive
permalink: /turkey-bowl/archive/
exclude: true
---

{% assign archive_pages = site.pages | where_exp: "page", "page.url contains '/turkey-bowl/archive/'" | where_exp: "page", "page.url != '/turkey-bowl/archive/'" | sort: "title" | reverse %}

<ul class="archive-list">
{% for archive_page in archive_pages %}
  <li><a href="{{ archive_page.url }}">{{ archive_page.title }} Archived Results</a></li>
{% endfor %}
</ul>
