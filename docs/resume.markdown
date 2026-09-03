---
layout: page
title: Resume
permalink: /resume/
---

{% assign resume = site.data["resume"] %}

## Experience

{% for job in resume.experience %}
### {{ job.company }}
#### {{ job.role }} • <time>{{ job.period }}</time>

{{ job.description }}

<ul>
  {% for highlight in job.highlights %}
  <li>{{ highlight }}</li>
  {% endfor %}
</ul>
{% endfor %}

## Education

{% for school in resume.education %}
### {{ school.institution }}

#### {{ school.degree }} • <time>{{ school.period }}</time>

{{ school.description }}
{% endfor %}
