---
layout: page
---

<img src="/assets/images/profile_pic.jpg" alt="Logan Thomas profile picture" style="border-radius: 50%; display: block; margin-left: auto; margin-right: auto; width:50%">
<h1 style="text-align: center;">Logan Thomas</h1>
<h3 style="text-align: center;">Software Developer | Technical Trainer | Python Enthusiast</h3>

<br/>
I'm a data science software engineer at [Fullstory](https://www.fullstory.com/){:target="_blank"}, passionate about building software and sharing knowledge. This site showcases my history of teaching, talks and tutorials given, and a blog to share recent open source contributions or other personal software musings.

[Learn more about me →](/about/)

<hr class="section-divider">

<h3>Courses Taught To Date</h3>
Total Courses: 62 Total Students: 694

| Year   | Count   |
| ------ | ------- |
| 2025   |  1      |
| 2024   |  2      |
| 2023   | 25      |
| 2022   | 20      |
| 2021   | 14      |


| Topic                                               | Count   |
| --------------------------------------------------- | ------- |
| Deep Learning (`tensorflow`, `keras`, `torch`)      | 19      |
| Machine Learning (`sklearn`)                        | 14      |
| Python Foundations (`numpy` & `pandas`)             | 13      |
| Data Analytics (`pandas` & `xarray`)                | 8       |
| Corporate Hackathon                                 | 4       |
| SciPy Tutorial (`numpy`)                            | 2       |
| Software Engineering (`unittest`, `logging`, `git`) | 2       |

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
