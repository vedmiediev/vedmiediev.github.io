---
layout: page
title: Sergii Vedmiediev's Blog
tagline: Supporting tagline
---
{% include JB/setup %}

<ul class="posts">
  {% for post in site.posts %}
  <div>
    <p><span>{{ post.date | date_to_string }}</span></p> 
	<h1> <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></h1>
	<p>
	 {{ post.excerpt | strip_html }}
	 {% if post.excerpt %}
	 <a href="{{ BASE_PATH }}{{ post.url }}">Read More</a>
	 {% endif %}
	</p>
	</div>
  {% endfor %}
</ul>


