---
permalink: /
title: "ArunGo'Log"
excerpt: "About me"
author_profile: true
redirect_from: 
  - /about/
  - /about.html
---

Sharing my learning and thougths about Architecture, Cloud, AI, Data Science

{% include base_path %} {% capture written_year %}'None'{% endcapture %} {% for post in site.posts %} {% capture year %}{{ post.date | date: '%Y' }}{% endcapture %} {% if year != written_year %}

## {{ year }}

{% capture written_year %}{{ year }}{% endcapture %} {% endif %} {% include archive-single.html %} {% endfor %}
