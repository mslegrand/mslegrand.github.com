---
layout: page
title: A Record of my Random Bio-Neural Activations
tagline: Supporting tagline
---
{% include JB/setup %}


## Introduction

This blog consists of a collection of post of some of my random observations, thoughts and insights. I hope some someone might find these useful.

### Post Listing:

<ul class="posts">
  {% for post in site.posts %}
    <li><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>

## To-Do

Bring coherence to the universe. 


