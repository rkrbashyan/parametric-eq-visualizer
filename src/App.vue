<template>
  <div class="app-container">
    <header>
      <h1>Parametric EQ Visualizer</h1>
      <p>
        Click a filter below to select it. Drag the points on the graph to
        adjust its properties.
      </p>
    </header>
    <main>
      <ParametricEQ v-model="activeFilters" :selected-id="selectedFilterId" />
      <div class="channel-switcher">
        <button
          :class="{ active: activeChannel === 'left' }"
          @click="activeChannel = 'left'"
        >
          Left Channel
        </button>
        <button
          :class="{ active: activeChannel === 'right' }"
          @click="activeChannel = 'right'"
        >
          Right Channel
        </button>
      </div>
      <div class="controls">
        <h2>{{ activeChannel }} Channel Filters</h2>
        <div class="filter-list">
          <div
            v-for="filter in activeFilters"
            :key="filter.id"
            class="filter-item"
            :class="{ selected: filter.id === selectedFilterId }"
            @click="selectedFilterId = filter.id"
          >
            <div class="filter-type">{{ filter.type }}</div>
            <div class="filter-prop-group" v-if="filter.type !== 'custom'">
              <div class="filter-prop">
                <span>Freq</span>
                <strong>{{ filter.hz.toFixed(0) }} Hz</strong>
              </div>
              <div class="filter-prop">
                <span>Gain</span>
                <strong>{{ filter.db.toFixed(1) }} dB</strong>
              </div>
              <div class="filter-prop">
                <span>Q</span>
                <strong>{{ filter.q.toFixed(2) }}</strong>
              </div>
            </div>
            <div class="filter-coeff-group" v-else>
              <div class="coeff-input">
                <label>b0</label>
                <input
                  type="number"
                  v-model.number="filter.b0"
                  @input="validateCoeffs(filter)"
                  step="0.01"
                />
              </div>
              <div class="coeff-input">
                <label>b1</label>
                <input
                  type="number"
                  v-model.number="filter.b1"
                  @input="validateCoeffs(filter)"
                  step="0.01"
                />
              </div>
              <div class="coeff-input">
                <label>b2</label>
                <input
                  type="number"
                  v-model.number="filter.b2"
                  @input="validateCoeffs(filter)"
                  step="0.01"
                />
              </div>
              <div class="coeff-input">
                <label>a1</label>
                <input
                  type="number"
                  v-model.number="filter.a1"
                  @input="validateCoeffs(filter)"
                  step="0.01"
                />
              </div>
              <div class="coeff-input">
                <label>a2</label>
                <input
                  type="number"
                  v-model.number="filter.a2"
                  @input="validateCoeffs(filter)"
                  step="0.01"
                />
              </div>
            </div>
          </div>
        </div>
      </div>
    </main>
  </div>
</template>

<script setup>
import { ref, computed } from 'vue';
import ParametricEQ from './components/ParametricEQ.vue';

// Main state now holds separate filter arrays for each channel
const channels = ref({
  left: [
    { id: 1, type: 'peaking', hz: 500, db: -4.5, q: 2.5 },
    { id: 2, type: 'low-shelf', hz: 120, db: 6.0, q: 0.707 },
    { id: 3, type: 'high-shelf', hz: 4000, db: 8.0, q: 0.707 },
    { id: 4, type: 'custom', b0: 1, b1: -1.92, b2: 0.99, a1: -1.85, a2: 0.95 },
  ],
  right: [
    { id: 100, type: 'peaking', hz: 100, db: -10.5, q: 2.5 },
    { id: 101, type: 'low-shelf', hz: 1200, db: 16.0, q: 0.707 },
    { id: 102, type: 'high-shelf', hz: 5000, db: 18.0, q: 0.707 },
    {
      id: 103,
      type: 'custom',
      b0: 1,
      b1: -1.92,
      b2: 0.99,
      a1: -1.85,
      a2: 0.95,
    },
  ],
});

const activeChannel = ref('left');
const selectedLeftFilterId = ref(1); // Default selected filter
const selectedRightFilterId = ref(100); // Default selected filter

// Computed property to get the filters for the currently active channel
const activeFilters = computed({
  get: () => channels.value[activeChannel.value],
  set: (newFilters) => {
    channels.value[activeChannel.value] = newFilters;
  },
});

const selectedFilterId = computed({
  get: () =>
    activeChannel.value === 'left'
      ? selectedLeftFilterId.value
      : selectedRightFilterId.value,
  set: (newId) => {
    if (activeChannel.value === 'left') {
      selectedLeftFilterId.value = newId;
    } else {
      selectedRightFilterId.value = newId;
    }
  },
});

function validateCoeffs(filter) {
  // Basic validation to ensure values are numbers.
  for (const key of ['b0', 'b1', 'b2', 'a1', 'a2']) {
    if (typeof filter[key] !== 'number' || isNaN(filter[key])) {
      filter[key] = 0;
    }
  }
}
</script>

<style>
:root {
  --bg-color: #f9fafb;
  --surface-color: #ffffff;
  --primary-color: #e11e4a;
  --text-color: #111827;
  --text-muted-color: #6b7280;
  --border-color: #e5e7eb;
}

body {
  font-family: 'Inter', sans-serif;
  background-color: var(--bg-color);
  color: var(--text-color);
  margin: 0;
  padding: 1rem;
}

.app-container {
  max-width: 900px;
  margin: 2rem auto;
  display: flex;
  flex-direction: column;
  gap: 2rem;
}

header {
  text-align: center;
}

h1 {
  margin: 0;
  font-size: 2.25rem;
}

p {
  color: var(--text-muted-color);
  font-size: 1.125rem;
}

main {
  background-color: var(--surface-color);
  border-radius: 1rem;
  border: 1px solid var(--border-color);
  overflow: hidden;
  box-shadow: 0 4px 6px -1px rgb(0 0 0 / 0.1), 0 2px 4px -2px rgb(0 0 0 / 0.1);
}

.channel-switcher {
  display: flex;
  border-bottom: 1px solid var(--border-color);
}

.channel-switcher button {
  flex-grow: 1;
  padding: 1rem;
  border: none;
  background-color: transparent;
  color: var(--text-muted-color);
  font-size: 1rem;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.2s ease;
  border-bottom: 3px solid transparent;
}

.channel-switcher button.active {
  color: var(--primary-color);
  border-bottom-color: var(--primary-color);
}

.controls {
  padding: 1.5rem;
  background-color: #f9fafb;
}

.controls h2 {
  margin-top: 0;
  border-bottom: 1px solid var(--border-color);
  padding-bottom: 0.5rem;
  margin-bottom: 1rem;
  text-transform: capitalize;
}

.filter-list {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(220px, 1fr));
  gap: 1rem;
}

.filter-item {
  background-color: #ffffff;
  border-radius: 0.5rem;
  padding: 1rem;
  cursor: pointer;
  border: 1px solid var(--border-color);
  transition: border-color 0.2s ease, box-shadow 0.2s ease;
  display: flex;
  flex-direction: column;
  gap: 0.75rem;
}

.filter-item:hover {
  border-color: #d1d5db;
}

.filter-item.selected {
  border-color: var(--primary-color);
  box-shadow: 0 0 0 2px var(--primary-color);
}

.filter-type {
  text-align: center;
  font-weight: bold;
  text-transform: uppercase;
  color: var(--primary-color);
  letter-spacing: 0.05em;
  font-size: 0.8rem;
  margin-bottom: 0.5rem;
}

.filter-prop-group {
  display: flex;
  justify-content: space-around;
  gap: 0.5rem;
  flex-wrap: wrap;
}

.filter-coeff-group {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 1rem;
}

.coeff-input {
  display: flex;
  flex-direction: column;
  align-items: center;
}

.coeff-input label {
  font-size: 0.875rem;
  color: var(--text-muted-color);
}

.coeff-input input {
  width: 80%;
  background-color: #f3f4f6;
  border: 1px solid var(--border-color);
  color: var(--text-color);
  border-radius: 0.25rem;
  text-align: center;
  padding: 0.25rem;
  -moz-appearance: textfield; /* Firefox */
}
.coeff-input input::-webkit-outer-spin-button,
.coeff-input input::-webkit-inner-spin-button {
  -webkit-appearance: none;
  margin: 0;
}

.filter-prop {
  text-align: center;
}

.filter-prop span {
  font-size: 0.875rem;
  color: var(--text-muted-color);
}

.filter-prop strong {
  display: block;
  font-size: 1.125rem;
  color: var(--text-color);
}
</style>
