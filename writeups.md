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
  display: flex;
  gap: 0.5rem;
  margin-bottom: 1rem;
  flex-wrap: wrap;
}
.search-input {
  flex: 1;
  min-width: 150px;
  padding: 0.375rem 0.5rem;
  border: 1px solid #ddd;
  border-radius: 4px;
  font-size: 0.875rem;
}
.filter-select {
  padding: 0.375rem 0.5rem;
  border: 1px solid #ddd;
  border-radius: 4px;
  font-size: 0.875rem;
  background: #fff;
}
.writeup-item {
  padding: 0.75rem;
  margin-bottom: 0.5rem;
  border: 1px solid #e0e0e0;
  border-radius: 4px;
}
.writeup-item.hidden {
  display: none;
}
.writeup-item h3 {
  margin: 0 0 0.25rem;
  font-size: 1.1rem;
}
.writeup-meta {
  display: flex;
  gap: 0.5rem;
  font-size: 0.75rem;
  color: #666;
  margin-bottom: 0.25rem;
}
.meta-difficulty, .meta-os {
  padding: 0.125rem 0.375rem;
  background: #eee;
  border-radius: 3px;
}
.writeup-summary {
  margin: 0;
  font-size: 0.875rem;
  color: #555;
}
.no-results {
  padding: 1rem;
  text-align: center;
  color: #666;
  background: #f9f9f9;
  border-radius: 4px;
  font-size: 0.875rem;
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