---
layout: page
---

<img src="/assets/images/profile_pic.jpg" style="border-radius: 50%; display: block; margin-left: auto; margin-right: auto; width:50%">
<h1 align="center">Logan Thomas</h1>
<h3 align="center">Software Developer | Technical Trainer | Python Enthusiast</h3>

<br/>
I currently work for [Enthought](https://www.enthought.com/){:target="_blank"} as a Scientific Software Developer and Technical Trainer. Previously, I worked for a protective design firm as a machine learning engineer and I've also been employed as a data scientist in the digital media industry.

I am passionate about software and love sharing my knowledge with others. This is my personal blog where I share helpful information about data science, machine learning, and python.

If you're interested in discussing any of the topics listed on this site, looking for training/mentorship, or just want to catch up over a cup of coffee (in-person or remote), feel free to reach out to me at [logan.thomas005@gmail.com](mailto:logan.thomas005@gmail.com?subject=[Website Contact])

To see other things I'm working on, check out my GitHub page: [loganthomas](https://github.com/loganthomas){:target="_blank"}
<br/><br/>
<h3>Courses Taught To Date</h3>
Total Courses: 55

| Year   | Count   |
| ------ | ------- |
| 2023   | 21      |
| 2022   | 20      |
| 2021   | 14      |


| Topic                                               | Count   |
| --------------------------------------------------- | ------- |
| Deep Learning (`tensorflow` & `keras`)              | 16      |
| Machine Learning (`sklearn`)                        | 13      |
| Python Foundations (`numpy` & `pandas`)             | 11      |
| Data Analytics (`pandas` & `xarray`)                | 8       |
| Corporate Hackathon                                 | 4       |
| SciPy Tutorial (`numpy`)                            | 2       |
| Software Engineering (`unnitest`, `logging`, `git`) | 1       |

<h3>Recent Posts</h3>
{%- if site.posts.size > 0 -%}
<ul class="post-list">
    {%- for post in site.posts limit:5-%}
    <li>
    {%- assign date_format = site.minima.date_format | default: "%b %-d, %Y" -%}
    <span class="post-meta">{{ post.date | date: date_format }}</span>
        <a class="post-link" href="{{ post.url | relative_url }}" style="font-size:20px">
        {{ post.title | escape }}
        </a>
    {%- if site.show_excerpts -%}
        {{ post.excerpt }}
    {%- endif -%}
    </li>
    {%- endfor -%}
</ul>
{%- endif -%}
