---
# You don't need to edit this file, it's empty on purpose.
# Edit theme's home layout instead if you wanna make some changes
# See: https://jekyllrb.com/docs/themes/#overriding-theme-defaults
layout: home
---
{% assign image_files = site.static_files %}
{% for myimage in image_files %}
    {{ myimage.path }}
{% endfor %}

endfor