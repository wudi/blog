“Make it correct, make it clear, make it concise, make it fast. In that order"

— Wes Dyer

## Notes

<ul>
  {% for post in site.posts %}
    <li>
      <a href="{{ post.url }}">{{ post.title }}</a>
    </li>
  {% endfor %}
</ul>
