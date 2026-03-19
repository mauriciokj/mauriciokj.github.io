---
layout: default
title: Blog
---

# Mauricio Krzesinski

Escrevendo sobre software, apps mobile, debugging, produto e as cicatrizes de guerra que o iOS deixa no caminho.

Bem-vindo ao meu canto da internet. Aqui eu publico aprendizados técnicos, bugs chatos que viraram case e anotações que talvez poupem algumas horas da sua vida.

## Posts

<ul class="post-list">
  {% for post in site.posts %}
    <li>
      <a href="{{ post.url | relative_url }}">{{ post.title }}</a>
      <span class="post-meta">— {{ post.date | date: "%d/%m/%Y" }}</span>
    </li>
  {% endfor %}
</ul>
