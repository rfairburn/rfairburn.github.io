---
layout: page
title: Devop Ninja
tagline: Randomly making things work with blood, sweat, and dirty hacks
---
{% include JB/setup %}

  {% for post in site.posts limit:5 %}
### [{{ post.title }}]({{ BASE_PATH }}{{ post.url }})
{{ post.date | date_to_string }}
{{ post.excerpt }}

<a href="{{ BASE_PATH }}{{ post.url }}">Read More</a>


  {% endfor %}

