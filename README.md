{% for post in site.posts %}

## <a href="{{ post.url }}">{{ post.title }}</a>
#### {{ post.date }}
> {{ post.excerpt }}

{% endfor %}
