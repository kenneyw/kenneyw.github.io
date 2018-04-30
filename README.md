  {% for post in site.posts %}
      <h2><a href="{{ post.url }}">{{ post.title }}</a></h2>
      <h4>{{ post.date }}</h4>
      {{ post.excerpt }}...
  {% endfor %}
