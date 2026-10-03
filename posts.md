---
layout: page
title: "Artigos por tema"
lead: "Todos os artigos do Super Simples, organizados por assunto."
permalink: /posts/
main_nav: false
---

{% assign cats = site.categories | sort %}
{% for category in cats %}
{% assign cat = category | first %}
<section class="topic-block" id="{{ cat }}">
  <h2><a href="{{ site.baseurl }}/category/{{ cat }}/">{% include cat-label.html cat=cat %}</a> <span class="topic-block__count">{{ category | last | size }}</span></h2>
  {% for d in site.descriptions %}{% if d.cat == cat %}<p class="topic-block__desc">{{ d.desc }}</p>{% endif %}{% endfor %}
  <ul class="post-index">
  {% for post in site.categories[cat] %}
    <li>
      <a href="{{ post.url | prepend: site.baseurl }}">{{ post.title }}</a>
      <time datetime="{{ post.date | date_to_xmlschema }}">{{ post.date | date: "%d/%m/%Y" }}</time>
    </li>
  {% endfor %}
  </ul>
</section>
{% endfor %}
