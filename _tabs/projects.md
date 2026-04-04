---
layout: page
icon: fas fa-folder-open
order: 1
excerpt: ""
---

{% for project in site.projects %}
## [{{ project.title }}]({{ project.url }})

{{ project.description | default: project.excerpt }}

---
{% endfor %}
