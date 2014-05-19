---
layout: page
title: ~jyu/notes
tagline: Supporting tagline
---
{% include JB/setup %}

<div style="width: 450px;">
<img src="{{ ASSET_PATH }}hyde/images/index/buns_01.jpg" style="width: 100%">
<img src="{{ ASSET_PATH }}hyde/images/index/buns_02.jpg" style="width: 100%">
<br />

Hello!

My name is Jessica. I'm a 4th year CS major at UC Berkeley.  
This is where you'll find my notes, ramblings, and random tirades.
</div>


### Contact
_email_ jyu '\0x40' cowsay.org  
_url_ [cowsay.org](http://cowsay.org/)

### Recent

<ul class="posts">  
{% for post in site.posts limit:20 %}  
<li>  
<span>{{ post.date | date_to_string }}</span> &raquo;  
<a href="{{ BASE_PATH }}{{ post.url }}">  
{{ post.title }}</a>  
</li>  
{% endfor %}  
</ul>

