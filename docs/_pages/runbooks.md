---
layout: page
title: "Runbooks"
permalink: /runbooks
---

# Runbooks

Following these /should/ yield a complete outcome.

<ul>
  {% assign runbook_posts = site.categories.runbooks %}
  {% for post in runbook_posts %}
    <li><a href="{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>
