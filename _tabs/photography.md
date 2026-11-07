---
# the default layout is 'page'
icon: fas fa-camera
order: 5
---

{% comment %}
  Each subfolder of imageshowcase/ becomes a section, ordered by its newest
  photo. Photos directly in imageshowcase/ come first, with no heading.
  Deeper folders are grouped under their top-level subfolder.
{% endcomment %}

{% assign photos = site.data.photos %}

{% if photos and photos.size > 0 %}
{% assign groups = '' | split: '' | push: '' %}
{% for key in photos %}
  {% assign parts = key | split: '/' %}
  {% if parts.size > 2 %}{% assign groups = groups | push: parts[1] %}{% endif %}
{% endfor %}
{% assign groups = groups | uniq %}

{% assign filters = '' %}
{% capture sections %}
{% for group in groups %}
  {% assign count = 0 %}
  {% capture tiles %}
  {% for key in photos %}
    {% assign parts = key | split: '/' %}
    {% assign folder = '' %}
    {% if parts.size > 2 %}{% assign folder = parts[1] %}{% endif %}
    {% if folder == group %}
      {% assign count = count | plus: 1 %}
      {% assign name = key | split: '/' | last | split: '.' | first | replace: '-', ' ' | replace: '_', ' ' %}
      {% comment %} No <img> here: the script below adds it as the tile nears the viewport {% endcomment %}
      <a class="popup img-link shimmer" href="{{ site.cdn }}/{{ key | uri_escape }}" aria-label="{{ name | escape }}"></a>
    {% endif %}
  {% endfor %}
  {% endcapture %}

  {% if count > 0 %}
  {% assign slug = '' %}
  {% if group != '' %}{% assign slug = group | slugify %}{% endif %}
  <section class="photo-section" data-group="{{ slug }}">
    {% if group != '' %}
    {% assign label = group | replace: '-', ' ' | replace: '_', ' ' %}
    {% assign rest = label | slice: 1, label.size %}
    {% capture title %}{{ label | slice: 0 | upcase | escape }}{{ rest | escape }}{% endcapture %}
    {% capture filter %}<a class="photo-filter" href="#{{ slug }}" data-filter="{{ slug }}">{{ title }} <small>{{ count }}</small></a>{% endcapture %}
    {% assign filters = filters | append: filter %}
    <h2>{{ title }} <small>{{ count }}</small></h2>
    {% endif %}
    <div class="photo-grid">{{ tiles }}</div>
  </section>
  {% endif %}
{% endfor %}
{% endcapture %}

{% if groups.size > 1 %}
<nav class="photo-filters" aria-label="Photo folders"><a class="photo-filter" href="#" data-filter="">All <small>{{ photos.size }}</small></a>{{ filters }}</nav>
{% endif %}

{{ sections }}
{% else %}
<p>No photos yet. Check back soon.</p>
{% endif %}

<style>
  .photo-filters {
    display: flex;
    flex-wrap: wrap;
    gap: 8px;
    margin-bottom: 1.75rem;
  }
  .content .photo-filters a.photo-filter,
  .content .photo-filters a.photo-filter:hover {
    padding: 0.3rem 0.85rem;
    border: 1px solid var(--main-border-color);
    border-radius: 999px;
    color: var(--text-color);
    font-size: 0.9rem;
    text-decoration: none;
    transition: none;
  }
  .content .photo-filters a.photo-filter:hover {
    border-color: var(--link-color);
    color: var(--link-color) !important; /* the theme's hover color is !important */
  }
  .content .photo-filters a.photo-filter[aria-current='true'] {
    border-color: var(--link-color);
    color: var(--link-color);
  }
  .photo-filter small {
    color: var(--text-muted-color);
  }
  /* Spacing only after a visible section, so filtering leaves no top gap */
  .photo-section:not([hidden]) ~ .photo-section {
    margin-top: 2.5rem;
  }
  .photo-section h2 {
    margin: 0 0 1rem;
  }
  .photo-section h2 small {
    color: var(--text-muted-color);
    font-size: 0.65em;
    font-weight: 400;
  }
  .photo-grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(150px, 1fr));
    gap: 8px;
  }
  .photo-grid > a {
    display: block;
    position: relative;
    aspect-ratio: 1;
    overflow: hidden;
    border-radius: 8px;
  }
  /* Must outrank Chirpy's `.content a.popup` margin, or row gaps double up */
  .content .photo-grid > a.popup {
    margin: 0;
  }
  /* Larger lead photo, only when enough tiles follow to fill in beside it */
  .photo-grid > a:first-child:nth-last-child(n + 3) {
    grid-column: span 2;
    grid-row: span 2;
  }
  /* Out of flow so a tall photo can't stretch its grid rows */
  .photo-grid img {
    position: absolute;
    inset: 0;
    display: block;
    width: 100%;
    height: 100%;
    margin: 0;
    border-radius: 0;
    object-fit: cover;
    transition: transform 0.35s ease;
  }
  .photo-grid > a:hover img,
  .photo-grid > a:focus-visible img {
    transform: scale(1.04);
  }
  @media (min-width: 768px) {
    .photo-grid {
      grid-template-columns: repeat(auto-fill, minmax(210px, 1fr));
      gap: 10px;
    }
  }
  @media (prefers-reduced-motion: reduce) {
    .photo-grid img {
      transition: none;
    }
  }
</style>

<script>
  (function () {
    var sections = Array.from(document.querySelectorAll('.photo-section'));
    var filters = Array.from(document.querySelectorAll('.photo-filter'));

    /* Keep lightbox arrows inside the current folder */
    sections.forEach(function (section, index) {
      section.querySelectorAll('a.popup').forEach(function (link) {
        link.setAttribute('data-gallery', 'photos-' + index);
      });
    });

    function applyFilter() {
      var slug = location.hash.slice(1);
      var known = sections.some(function (section) {
        return section.dataset.group === slug;
      });
      if (!known) slug = '';

      sections.forEach(function (section) {
        section.hidden = slug !== '' && section.dataset.group !== slug;
      });
      filters.forEach(function (filter) {
        filter.setAttribute('aria-current', String(filter.dataset.filter === slug));
      });
    }

    window.addEventListener('hashchange', applyFilter);
    applyFilter();

    /* Native loading="lazy" fetches far ahead of the viewport, so load by hand.
       Tiles in hidden sections never intersect, so they stay unloaded. */
    function loadPhoto(link) {
      var photo = new Image();
      photo.alt = link.getAttribute('aria-label');
      photo.decoding = 'async';
      photo.onload = photo.onerror = function () {
        link.classList.remove('shimmer');
      };
      photo.src = link.href;
      link.appendChild(photo);
    }

    var loader = new IntersectionObserver(
      function (entries) {
        entries.forEach(function (entry) {
          if (!entry.isIntersecting) return;
          loader.unobserve(entry.target);
          loadPhoto(entry.target);
        });
      },
      { rootMargin: '200px 0px' }
    );

    document.querySelectorAll('.photo-grid > a').forEach(function (link) {
      loader.observe(link);
    });
  })();
</script>
