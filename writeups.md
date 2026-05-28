---
layout: single
title: "Writeups"
permalink: /writeups/
---

<div class="writeups-filters">
  <input type="text" id="writeup-search" placeholder="Buscar..." class="search-input">
  <select id="difficulty-filter" class="filter-select">
    <option value="">Dificultad</option>
    <option value="fácil">Fácil</option>
    <option value="medio">Medio</option>
    <option value="difícil">Difícil</option>
  </select>
  <select id="os-filter" class="filter-select">
    <option value="">Sistema</option>
    <option value="linux">Linux</option>
    <option value="windows">Windows</option>
  </select>
</div>

<div class="writeups-list" id="writeups-list">
{% for writeup in site.writeups %}
  <div class="writeup-item" 
       data-title="{{ writeup.title | downcase }}" 
       data-tags="{{ writeup.tags | join: ' ' | downcase }}"
       data-difficulty="{{ writeup.difficulty | downcase }}" 
       data-os="{{ writeup.operating_system | downcase }}">
    <h3>
      <a href="{{ writeup.url }}">{{ writeup.title }}</a>
    </h3>
    <div class="writeup-meta">
      <span class="meta-difficulty">{{ writeup.difficulty }}</span>
      <span class="meta-os">{{ writeup.operating_system }}</span>
      {% if writeup.date %}
      <span class="meta-date">{{ writeup.date | date: "%d/%m/%Y" }}</span>
      {% endif %}
    </div>
    {% if writeup.tags %}
    <div class="writeup-tags">
      {% for tag in writeup.tags %}
      <span class="tag-pill">{{ tag }}</span>
      {% endfor %}
    </div>
    {% endif %}
  </div>
{% endfor %}
</div>

<div class="no-results" id="no-results" style="display: none;">
  <p>No se encontraron writeups con los filtros seleccionados.</p>
</div>
<script>
document.addEventListener('DOMContentLoaded', function() {
  const searchInput = document.getElementById('writeup-search');
  const difficultyFilter = document.getElementById('difficulty-filter');
  const osFilter = document.getElementById('os-filter');
  const writeupItems = document.querySelectorAll('.writeup-item');
  const noResults = document.getElementById('no-results');

  function filterWriteups() {
    const searchTerm = searchInput.value.toLowerCase();
    const difficulty = difficultyFilter.value.toLowerCase();
    const os = osFilter.value.toLowerCase();
    
    let visibleCount = 0;

    writeupItems.forEach(item => {
      const title = item.dataset.title;
      const tags = item.dataset.tags || '';
      const itemDifficulty = item.dataset.difficulty || '';
      const itemOs = item.dataset.os || '';

      const matchesSearch = !searchTerm || title.includes(searchTerm) || tags.includes(searchTerm);
      const matchesDifficulty = !difficulty || itemDifficulty === difficulty;
      const matchesOs = !os || itemOs.includes(os);

      if (matchesSearch && matchesDifficulty && matchesOs) {
        item.classList.remove('hidden');
        visibleCount++;
      } else {
        item.classList.add('hidden');
      }
    });

    noResults.style.display = visibleCount === 0 ? 'block' : 'none';
  }

  searchInput.addEventListener('input', filterWriteups);
  difficultyFilter.addEventListener('change', filterWriteups);
  osFilter.addEventListener('change', filterWriteups);
});
</script>