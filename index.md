<ul>
  {% for post in site.posts %}
    <li class="post">
      {{ post.date | date: "%m/%Y" }} - <a href="{{ post.url }}">{{ post.title }}</a>
    </li>
  {% endfor %}
</ul>
