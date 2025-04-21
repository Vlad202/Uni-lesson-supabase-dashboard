// App.vue з покращеним інтерфейсом: фільтри, додавання, оновлений вигляд
<template>
  <div class="container">
    <header class="header">
      <h1>📊 Екологічний дашборд</h1>
      <p>Фільтрація, додавання та перегляд екологічних спостережень</p>
    </header>

    <!-- 🔍 Фільтри -->
    <section class="filters">
      <select v-model="filters.station">
        <option value="">Усі станції</option>
        <option v-for="s in stations" :key="s.id" :value="s.name">{{ s.name }}</option>
      </select>
      <select v-model="filters.indicator">
        <option value="">Усі показники</option>
        <option v-for="i in indicators" :key="i.id" :value="i.name">{{ i.name }}</option>
      </select>
      <input type="date" v-model="filters.date" />
      <button @click="applyFilters">Застосувати</button>
    </section>

    <!-- ➕ Додавання нового спостереження -->
    <section class="add-form">
      <h2>➕ Додати нове спостереження</h2>
      <form @submit.prevent="addObservation">
        <select v-model="newEntry.station_id" required>
          <option value="">Оберіть станцію</option>
          <option v-for="s in stations" :key="s.id" :value="s.id">{{ s.name }}</option>
        </select>
        <select v-model="newEntry.indicator_id" required>
          <option value="">Оберіть показник</option>
          <option v-for="i in indicators" :key="i.id" :value="i.id">{{ i.name }}</option>
        </select>
        <input type="number" step="any" placeholder="Значення" v-model="newEntry.value" required />
        <button type="submit">Додати</button>
      </form>
    </section>

    <!-- 🔢 Метрики -->
    <section class="metrics">
      <div v-for="m in metrics" :key="m.label" class="card">
        <div class="label">{{ m.label }}</div>
        <div class="value">{{ m.value }}</div>
      </div>
    </section>

    <!-- 📈 Графік -->
    <section class="chart">
      <h2>📈 Графік температури</h2>
      <canvas ref="chartCanvas" height="100"></canvas>
    </section>

    <!-- 📋 Таблиця -->
    <section class="table-section">
      <h2>📋 Останні спостереження</h2>
      <div class="table-container">
        <table>
          <thead>
            <tr>
              <th>Станція</th>
              <th>Показник</th>
              <th>Значення</th>
              <th>Дата</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="o in observations" :key="o.id">
              <td>{{ o.station }}</td>
              <td>{{ o.indicator }}</td>
              <td>{{ o.value }}</td>
              <td>{{ o.date }}</td>
            </tr>
          </tbody>
        </table>
      </div>
    </section>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue'
import { createClient } from '@supabase/supabase-js'
import { Chart, registerables } from 'chart.js'
Chart.register(...registerables)

const supabase = createClient(import.meta.env.VITE_SUPABASE_URL, import.meta.env.VITE_SUPABASE_KEY)

const observations = ref([])
const stations = ref([])
const indicators = ref([])
const metrics = ref([
  { label: 'Середня температура', value: '—' },
  { label: 'CO₂ сьогодні', value: '—' },
  { label: 'Шум сьогодні', value: '—' },
  { label: 'Активні станції', value: '—' },
])

const filters = ref({ station: '', indicator: '', date: '' })
const newEntry = ref({ station_id: '', indicator_id: '', value: '' })
const chartCanvas = ref(null)
let chart

const loadObservations = async () => {
  const { data: obs } = await supabase.from('observations').select(`id, value, measured_at, indicator_id, indicators(name), observationstations(name)`).order('measured_at', { ascending: false }).limit(100)
  observations.value = obs.map(o => ({
    id: o.id,
    value: o.value,
    date: new Date(o.measured_at).toLocaleString(),
    indicator: o.indicators.name,
    station: o.observationstations.name,
    station_id: o.observationstations.id,
    indicator_id: o.indicator_id,
    raw: o
  }))
  updateChart()
  updateMetrics()
}

const updateChart = () => {
  const tempData = observations.value.filter(o => o.indicator === 'Температура').reverse()
  if (chart) chart.destroy()
  chart = new Chart(chartCanvas.value, {
    type: 'line',
    data: {
      labels: tempData.map(o => new Date(o.raw.measured_at).toLocaleTimeString()),
      datasets: [{
        label: 'Температура',
        data: tempData.map(o => o.value),
        borderColor: '#007acc',
        backgroundColor: 'rgba(0, 122, 204, 0.1)',
        borderWidth: 2,
        tension: 0.4
      }]
    },
    options: {
      responsive: true,
      plugins: { legend: { display: true } },
      scales: { y: { beginAtZero: true } }
    }
  })
}

const updateMetrics = () => {
  const tempValues = observations.value.filter(o => o.indicator === 'Температура').map(o => o.value)
  if (tempValues.length) metrics.value[0].value = (tempValues.reduce((a,b) => a+b,0) / tempValues.length).toFixed(1)
  const co2 = observations.value.find(o => o.indicator === 'CO₂')
  const noise = observations.value.find(o => o.indicator === 'Рівень шуму')
  if (co2) metrics.value[1].value = co2.value
  if (noise) metrics.value[2].value = noise.value
  metrics.value[3].value = [...new Set(observations.value.map(o => o.station))].length
}

const applyFilters = () => {
  loadObservations()
  if (filters.value.station) observations.value = observations.value.filter(o => o.station === filters.value.station)
  if (filters.value.indicator) observations.value = observations.value.filter(o => o.indicator === filters.value.indicator)
  if (filters.value.date) observations.value = observations.value.filter(o => o.date.startsWith(new Date(filters.value.date).toLocaleDateString()))
  updateChart()
  updateMetrics()
}

const addObservation = async () => {
  await supabase.from('observations').insert({
    station_id: newEntry.value.station_id,
    indicator_id: newEntry.value.indicator_id,
    value: parseFloat(newEntry.value.value),
    measured_at: new Date().toISOString()
  })
  newEntry.value = { station_id: '', indicator_id: '', value: '' }
  await loadObservations()
}

onMounted(async () => {
  const { data: s } = await supabase.from('observationstations').select()
  const { data: i } = await supabase.from('indicators').select()
  stations.value = s || []
  indicators.value = i || []
  await loadObservations()
})
</script>

<style>
body {
  margin: 0;
  font-family: Arial, sans-serif;
  background-color: #eef5f9;
}
.container {
  padding: 2rem;
  max-width: 1200px;
  margin: auto;
}
.header h1 {
  font-size: 2rem;
  color: #0b3d91;
  margin-bottom: 0.25rem;
}
.header p {
  color: #333;
  margin-bottom: 1.5rem;
}
.filters, .add-form form {
  display: flex;
  gap: 1rem;
  flex-wrap: wrap;
  margin-bottom: 1.5rem;
  align-items: center;
}
select, input[type="number"], input[type="date"] {
  padding: 0.4rem 0.6rem;
  border-radius: 4px;
  border: 1px solid #ccc;
  min-width: 150px;
}
button {
  background-color: #007acc;
  color: white;
  border: none;
  padding: 0.5rem 1rem;
  border-radius: 4px;
  cursor: pointer;
}
button:hover {
  background-color: #005fa3;
}
.metrics {
  display: flex;
  flex-wrap: wrap;
  gap: 1rem;
  margin-bottom: 2rem;
}
.card {
  flex: 1 1 200px;
  background: white;
  border-left: 5px solid #007acc;
  border-radius: 0.5rem;
  padding: 1rem;
  box-shadow: 0 2px 5px rgba(0,0,0,0.05);
}
.label {
  font-size: 0.8rem;
  color: #555;
}
.value {
  font-size: 2rem;
  color: #0b3d91;
  font-weight: bold;
}
.chart {
  margin-bottom: 2rem;
  background: white;
  padding: 1rem;
  border-radius: 0.5rem;
  box-shadow: 0 2px 5px rgba(0,0,0,0.05);
}
.table-section h2 {
  margin-bottom: 1rem;
}
.table-container {
  overflow-x: auto;
  background: white;
  border-radius: 0.5rem;
  box-shadow: 0 2px 5px rgba(0,0,0,0.05);
}
table {
  width: 100%;
  border-collapse: collapse;
}
th, td {
  padding: 0.75rem;
  text-align: left;
  border-bottom: 1px solid #ddd;
}
th {
  background-color: #cfe7ff;
  color: #0b3d91;
}
tr:hover {
  background-color: #f1f9ff;
}
</style>