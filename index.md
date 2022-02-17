---
# Feel free to add content and custom Front Matter to this file.
# To modify the layout, see https://jekyllrb.com/docs/themes/#overriding-theme-defaults

layout: default
---

# Welcome stranger

17/02/2022

Was a fucking troubling period of my life.

I'll be back soon with new cool stuff  
stay tuned  


yours hackersly,  
k0z4c

## HackTheBox
<ul>
  {% for post in site.posts %}
    <li>
      <a href="{{ post.url }}">{{ post.title }}</a>
    </li>
  {% endfor %}
</ul>

[here]({% link about.md  %}) some info about me.

![hacking]({% link assets/hacking_kittens.jpg %})  
