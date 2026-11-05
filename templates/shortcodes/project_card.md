{% component project_card() %}
<div class="project-card">
    {{ body | markdown(inline=true) | safe }}
</div>
{% endcomponent project_card %}
