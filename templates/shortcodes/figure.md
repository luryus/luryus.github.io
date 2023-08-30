<figure>
    <img src="{{ src }}" alt="{{ alt }}" title="{{body|markdown|striptags}}">
    <figcaption>
    {{ body | markdown(inline=true) | safe }}
    </figcaption>
</figure>