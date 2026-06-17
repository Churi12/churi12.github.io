---
layout: default
title: Home
---

# Miguel Santos

Infrastructure and platform engineering. I work mostly with Kubernetes, AWS, ArgoCD, Karpenter, and the Grafana stack. I write here about things I learned the hard way, usually while fixing them.

## Posts

<ul>
  {% for post in site.posts %}
    <li>
      <a href="{{ post.url | relative_url }}">{{ post.title }}</a>
      <span> &mdash; {{ post.date | date: "%Y-%m-%d" }}</span>
    </li>
  {% endfor %}
</ul>
