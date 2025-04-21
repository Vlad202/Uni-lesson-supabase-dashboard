// Основной файл App.vue без TailwindCSS, адаптований на простий CSS
<template>
  <div class="container">
    <header class="header">
      <h1>📊 Екологічний дашборд</h1>
      <p>Інтерактивна система моніторингу стану довкілля</p>
    </header>

    <section class="metrics">
      <div v-for="m in metrics" :key="m.label" class="card">
        <div class="label">{{ m.label }}</div>
        <div class="value">{{ m.value }}</div>
      </div>
    </section>

    <section class="chart">
      <h2>📈 Графік температури</h2>
      <canvas ref="chartCanvas" height="100"></canvas>
    </section>

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

const supabase = createClient(import.meta.env.VITE_SUPABASE_URL, import.meta.env.VITE_SUPABASE_ANON_KEY)

const metrics = ref([
  { label: 'Середня температура', value: '—' },
  { label: 'CO₂ сьогодні', value: '—' },
  { label: 'Шум сьогодні', value: '—' },
  { label: 'Активні станції', value: '—' },
])

const observations = ref([])
const chartCanvas = ref(null)

onMounted(async () => {
  const { data: obs } = await supabase.from('observations').select(`id, value, measured_at, indicator_id, indicators(name), observationstations(name)`).order('measured_at', { ascending: false }).limit(10)
  observations.value = obs.map(o => ({
    id: o.id,
    value: o.value,
    date: new Date(o.measured_at).toLocaleString(),
    indicator: o.indicators.name,
    station: o.observationstations.name
  }))

  const tempData = obs.filter(o => o.indicators.name === 'Температура').reverse()
  const chart = new Chart(chartCanvas.value, {
    type: 'line',
    data: {
      labels: tempData.map(o => new Date(o.measured_at).toLocaleTimeString()),
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
      scales: {
        y: {
          beginAtZero: true
        }
      }
    }
  })

  const { data: stations } = await supabase.from('observationstations').select()
  metrics.value[3].value = stations?.length || 0

  const tempValues = tempData.map(o => o.value)
  if (tempValues.length) metrics.value[0].value = (tempValues.reduce((a,b) => a+b,0) / tempValues.length).toFixed(1)

  const co2 = obs.find(o => o.indicators.name === 'CO₂')
  if (co2) metrics.value[1].value = co2.value
  const noise = obs.find(o => o.indicators.name === 'Рівень шуму')
  if (noise) metrics.value[2].value = noise.value
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
  margin-bottom: 0.2rem;
  color: #0b3d91;
}
.header p {
  color: #333;
  margin-bottom: 2rem;
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