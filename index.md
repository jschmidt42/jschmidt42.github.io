I am porting my [old site](http://www.equals-forty-two.com/) to this new github.io blogging platform.

I'll be moving all the content to this new web site. Once that is done, I'll re-use the domain name for this site.

{{ post.excerpt }}

<ul class="myposts">
{% for post in site.categories.podcasts limit:3 %}
    <li><a href="{{ post.url }}">{{ post.title}}</a>
    <span class="postDate">{{ post.date | date: "%b %-d, %Y" }}</span>
    </li>
{% endfor %}
