---
title: Linux
icon: fab fa-linux
order: 1
seccion: Linux
---

Todo empieza en la terminal. Aquí voy subiendo el wargame **Bandit** de OverTheWire nivel a nivel, explicando cada comando para que aprendas a moverte por Linux como en casa.

{% include lang.html %}
{% assign posts = site.categories[page.seccion] %}

{% if posts and posts.size > 0 %}
<div id="page-category">
  <ul class="content ps-0">
    {% for post in posts %}
      <li class="d-flex justify-content-between px-md-3">
        <a href="{{ post.url | relative_url }}">{{ post.title }}</a>
        <span class="dash flex-grow-1"></span>
        {% include datetime.html date=post.date class='text-muted small text-nowrap' lang=lang %}
      </li>
    {% endfor %}
  </ul>
</div>
{% else %}
> Todavía no hay nada por aquí. El primer post está en camino.
{: .prompt-info }
{% endif %}
