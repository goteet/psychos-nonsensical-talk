<ul>
{% for post in site.posts %}
  <li>
    <span>{{ post.date | date_to_string }}</span>
    &raquo; 
    <a href="{{ site.baseurl }}{{ post.url }}">{{ post.title }}</a>
  </li>
{% endfor %}
</ul>