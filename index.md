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
	<a href="{{ BASE_PATH }}{{ post.url }}#disqus_thread"></a>
	{% if post.content contains '<!--more-->' %}
		{{ post.content | split:'<!--more-->' | first | strip_html }}
		<a href="{{ BASE_PATH }}{{ post.url }}">Read More</a>
	{% else %}
		<!-- Case for when no excerpt is defined -->
	{% endif %}
	</p>
	</div>
  {% endfor %}
</ul>




