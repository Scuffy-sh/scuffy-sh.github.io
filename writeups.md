---
layout: single
title: "Writeups"
permalink: /writeups/
---

## Buscador y Filtros

<div class="writeups-filters">
  <div class="filter-row">
    <input type="text" id="writeup-search" placeholder="Buscar writeups..." class="search-input">
  </div>
  <div class="filter-row">
    <select id="difficulty-filter" class="filter-select">
      <option value="">Todas las dificultades</option>
      <option value="fácil">Fácil</option>
      <option value="medio">Medio</option>
      <option value="difícil">Difícil</option>
    </select>
    <select id="os-filter" class="filter-select">
      <option value="">Todos los sistemas</option>
      <option value="linux">Linux</option>
      <option value="windows">Windows</option>
    </select>
  </div>
</div>

---

## Lista de Writeups

<div class="writeups-list" id="writeups-list">
{% for writeup in site.writeups %}
  <div class="writeup-item" 
       data-title="{{ writeup.title | downcase }}" 
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
    {% if writeup.summary %}
    <p class="writeup-summary">{{ writeup.summary }}</p>
    {% endif %}
  </div>
{% endfor %}
</div>

<div class="no-results" id="no-results" style="display: none;">
  <p>No se encontraron writeups con los filtros seleccionados.</p>
</div>

<style>
.writeups-filters {
  margin-bottom: 2rem;
  padding: 1rem;
  background: var(--page-background);
  border-radius: 4px;
}
.filter-row {
  display: flex;
  gap: 1rem;
  margin-bottom: 0.5rem;
}
.filter-row:last-child {
  margin-bottom: 0;
}
.search-input, .filter-select {
  padding: 0.5rem;
  border: 1px solid #ccc;
  border-radius: 4px;
  font-size: 1rem;
}
.search-input {
  flex: 1;
}
.filter-select {
  min-width: 150px;
}
.writeup-item {
  padding: 1rem;
  margin-bottom: 1rem;
  border: 1px solid #e0e0e0;
  border-radius: 4px;
  transition: opacity 0.2s ease;
}
.writeup-item.hidden {
  display: none;
}
.writeup-item h3 {
  margin: 0 0 0.5rem;
}
.writeup-meta {
  display: flex;
  gap: 1rem;
  font-size: 0.875rem;
  color: #666;
  margin-bottom: 0.5rem;
}
.meta-difficulty, .meta-os {
  padding: 0.125rem 0.5rem;
  background: #eee;
  border-radius: 3px;
}
.meta-date {
  color: #888;
}
.writeup-summary {
  margin: 0;
  color: #555;
}
.no-results {
  padding: 2rem;
  text-align: center;
  color: #666;
  background: #f9f9f9;
  border-radius: 4px;
}
</style>

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
      const itemDifficulty = item.dataset.difficulty || '';
      const itemOs = item.dataset.os || '';

      const matchesSearch = !searchTerm || title.includes(searchTerm);
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