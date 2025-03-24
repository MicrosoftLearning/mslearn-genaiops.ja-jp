---
title: オンラインでホスティングされている手順
permalink: index.html
layout: home
---

# Microsoft Learn - ハンズオン演習

次のハンズオン演習は、[Microsoft Learn](https://docs.microsoft.com/training/) トレーニングをサポートするように設計されています。

{% assign labs = site.pages | where_exp:"page", "page.url contains '/Instructions'" %}
| |
| --- | --- | 
{% for activity in labs %}| [{{ activity.lab.title }}{% if activity.lab.type %} - {{ activity.lab.type }}{% endif %}]({{ site.github.url }}{{ activity.url }}) |
{% endfor %}
