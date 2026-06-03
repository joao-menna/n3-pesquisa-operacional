<script setup lang="ts">
import { computed, ref } from 'vue'
import {
  Chart as ChartJS,
  CategoryScale,
  LinearScale,
  BarElement,
  PointElement,
  LineElement,
  Tooltip,
  Legend,
} from 'chart.js'
import { Bar, Line } from 'vue-chartjs'

ChartJS.register(
  CategoryScale,
  LinearScale,
  BarElement,
  PointElement,
  LineElement,
  Tooltip,
  Legend,
)

const presets = [
  {
    name: "Base",
    numSimulations: 10_000,
    numWeeks: 52,
    initialInventory: 10,
    s: 4,
    q: 6,
    holdingCostPerUnit: 200,
    stockoutCostPerUnit: 1000,
    orderCostFixed: 500,
    poissonLambda: 3,
    histogramBins: 40,
    seed: 42,
  },
  {
    name: "Arriscado",
    numSimulations: 10_000,
    numWeeks: 52,
    initialInventory: 10,
    s: 4,
    q: 6,
    holdingCostPerUnit: 200,
    stockoutCostPerUnit: 1500,
    orderCostFixed: 500,
    poissonLambda: 4.5,
    histogramBins: 40,
    seed: 42,
  },
  {
    name: "Conservador",
    numSimulations: 10_000,
    numWeeks: 52,
    initialInventory: 10,
    s: 4,
    q: 6,
    holdingCostPerUnit: 200,
    stockoutCostPerUnit: 500,
    orderCostFixed: 500,
    poissonLambda: 1.5,
    histogramBins: 40,
    seed: 42,
  },
]

const numSimulations = ref(10_000)
const numWeeks = ref(52)
const initialInventory = ref(10)
const s = ref(4)
const q = ref(6)

const holdingCostPerUnit = ref(200)
const stockoutCostPerUnit = ref(1000)
const orderCostFixed = ref(500)
const poissonLambda = ref(3)

const seed = ref(42)
const histogramBins = ref(40)

const isRunning = ref(false)
const progress = ref(0)
const completedSimulations = ref(0)

const totalCosts = ref<number[]>([])
const serviceLevels = ref<number[]>([])
const runningMeans = ref<number[]>([])

function mulberry32(initialSeed: number) {
  let t = initialSeed >>> 0
  return () => {
    t += 0x6D2B79F5
    let x = Math.imul(t ^ (t >>> 15), t | 1)
    x ^= x + Math.imul(x ^ (x >>> 7), x | 61)
    return ((x ^ (x >>> 14)) >>> 0) / 4294967296
  }
}

function samplePoisson(lambda: number, random: () => number) {
  const l = Math.exp(-lambda)
  let p = 1
  let k = 0

  do {
    k += 1
    p *= random()
  } while (p > l)

  return k - 1
}

function sampleTriangular(min: number, mode: number, max: number, random: () => number) {
  const u = random()
  const c = (mode - min) / (max - min)

  if (u <= c) {
    return min + Math.sqrt(u * (max - min) * (mode - min))
  }

  return max - Math.sqrt((1 - u) * (max - min) * (max - mode))
}

function simulateBatch(
  batchSize: number,
  random: () => number,
  currentCosts: number[],
  currentServiceLevels: number[],
  currentRunningMean: number[],
) {
  for (let sim = 0; sim < batchSize; sim += 1) {
    let inventory = initialInventory.value
    let orderInTransit = false
    let deliveryWeek = -1

    let simHoldingCost = 0
    let simStockoutCost = 0
    let simOrderCost = 0
    let weeksWithFullService = 0

    for (let week = 1; week <= numWeeks.value; week += 1) {
      if (orderInTransit && week === deliveryWeek) {
        inventory += q.value
        orderInTransit = false
        deliveryWeek = -1
      }

      const demand = samplePoisson(poissonLambda.value, random)

      let stockout = 0
      if (inventory >= demand) {
        inventory -= demand
        weeksWithFullService += 1
      } else {
        stockout = demand - inventory
        inventory = 0
      }

      simHoldingCost += inventory * holdingCostPerUnit.value
      simStockoutCost += stockout * stockoutCostPerUnit.value

      if (inventory <= s.value && !orderInTransit) {
        orderInTransit = true
        const leadTime = Math.round(sampleTriangular(1, 2, 4, random))
        deliveryWeek = week + leadTime
        simOrderCost += orderCostFixed.value
      }
    }

    const totalCost = simHoldingCost + simStockoutCost + simOrderCost
    currentCosts.push(totalCost)
    currentServiceLevels.push(weeksWithFullService / numWeeks.value)

    const n = currentCosts.length
    const previousMean = n > 1 ? currentRunningMean[n - 2] : 0
    const newMean = previousMean + (totalCost - previousMean) / n
    currentRunningMean.push(newMean)
  }
}

function percentile(values: number[], p: number) {
  if (values.length === 0) return 0
  const sorted = [...values].sort((a, b) => a - b)
  const index = (p / 100) * (sorted.length - 1)
  const lower = Math.floor(index)
  const upper = Math.ceil(index)

  if (lower === upper) return sorted[lower]
  const weight = index - lower
  return sorted[lower] * (1 - weight) + sorted[upper] * weight
}

async function runSimulation() {
  if (isRunning.value) return

  isRunning.value = true
  progress.value = 0
  completedSimulations.value = 0

  const random = mulberry32(seed.value)
  const currentCosts: number[] = []
  const currentServiceLevels: number[] = []
  const currentRunningMean: number[] = []

  const chunkSize = Math.max(100, Math.floor(numSimulations.value / 40))

  while (currentCosts.length < numSimulations.value) {
    const remaining = numSimulations.value - currentCosts.length
    const batchSize = Math.min(chunkSize, remaining)

    simulateBatch(batchSize, random, currentCosts, currentServiceLevels, currentRunningMean)

    completedSimulations.value = currentCosts.length
    progress.value = (currentCosts.length / numSimulations.value) * 100

    totalCosts.value = [...currentCosts]
    serviceLevels.value = [...currentServiceLevels]
    runningMeans.value = [...currentRunningMean]

    await new Promise((resolve) => setTimeout(resolve, 0))
  }

  isRunning.value = false
}

const meanCost = computed(() => {
  if (totalCosts.value.length === 0) return 0
  const sum = totalCosts.value.reduce((acc, value) => acc + value, 0)
  return sum / totalCosts.value.length
})

const medianCost = computed(() => percentile(totalCosts.value, 50))
const p95Cost = computed(() => percentile(totalCosts.value, 95))

const meanServiceLevel = computed(() => {
  if (serviceLevels.value.length === 0) return 0
  const sum = serviceLevels.value.reduce((acc, value) => acc + value, 0)
  return (sum / serviceLevels.value.length) * 100
})

const histogramData = computed(() => {
  if (totalCosts.value.length === 0) {
    return {
      labels: [],
      datasets: [
        {
          label: 'Frequencia',
          data: [],
          backgroundColor: 'rgba(15, 118, 110, 0.7)',
          borderColor: 'rgba(13, 148, 136, 1)',
          borderWidth: 1,
        },
      ],
    }
  }

  const minCost = Math.min(...totalCosts.value)
  const maxCost = Math.max(...totalCosts.value)
  const binCount = Math.max(10, histogramBins.value)
  const binWidth = Math.max(1, (maxCost - minCost) / binCount)

  const counts = new Array(binCount).fill(0)

  for (const cost of totalCosts.value) {
    const rawIndex = Math.floor((cost - minCost) / binWidth)
    const index = Math.min(binCount - 1, Math.max(0, rawIndex))
    counts[index] += 1
  }

  const labels = counts.map((_, index) => {
    const start = minCost + index * binWidth
    const end = start + binWidth
    return `${Math.round(start / 1000)}k-${Math.round(end / 1000)}k`
  })

  return {
    labels,
    datasets: [
      {
        label: 'Distribuicao de custos anuais',
        data: counts,
        backgroundColor: 'rgba(15, 118, 110, 0.7)',
        borderColor: 'rgba(13, 148, 136, 1)',
        borderWidth: 1,
      },
    ],
  }
})

const runningMeanData = computed(() => {
  const step = Math.max(1, Math.floor(runningMeans.value.length / 120))
  const compactCosts: number[] = []
  const labels: string[] = []

  for (let i = 0; i < runningMeans.value.length; i += step) {
    compactCosts.push(runningMeans.value[i])
    labels.push(String(i + 1))
  }

  return {
    labels,
    datasets: [
      {
        label: 'Custo medio acumulado (R$)',
        data: compactCosts,
        borderColor: 'rgba(234, 88, 12, 1)',
        backgroundColor: 'rgba(234, 88, 12, 0.25)',
        pointRadius: 0,
        tension: 0.2,
      },
    ],
  }
})

const chartOptions = {
  responsive: true,
  maintainAspectRatio: false,
  plugins: {
    legend: {
      display: true,
    },
  },
}

const lineOptions = {
  ...chartOptions,
  scales: {
    x: {
      title: {
        display: true,
        text: 'Simulacoes processadas',
      },
    },
    y: {
      title: {
        display: true,
        text: 'R$',
      },
    },
  },
}

const histogramOptions = {
  ...chartOptions,
  scales: {
    x: {
      ticks: {
        maxRotation: 0,
        autoSkip: true,
        maxTicksLimit: 10,
      },
      title: {
        display: true,
        text: 'Faixa de custo anual',
      },
    },
    y: {
      title: {
        display: true,
        text: 'Frequencia',
      },
    },
  },
}

function currency(value: number) {
  return value.toLocaleString('pt-BR', {
    style: 'currency',
    currency: 'BRL',
    maximumFractionDigits: 0,
  })
}

function setPreset(preset: typeof presets[number]) {
  numSimulations.value = preset.numSimulations
  numWeeks.value = preset.numWeeks
  initialInventory.value = preset.initialInventory
  s.value = preset.s
  q.value = preset.q
  holdingCostPerUnit.value = preset.holdingCostPerUnit
  stockoutCostPerUnit.value = preset.stockoutCostPerUnit
  orderCostFixed.value = preset.orderCostFixed
  poissonLambda.value = preset.poissonLambda
  histogramBins.value = preset.histogramBins
  seed.value = preset.seed
}

</script>

<template>
  <div class="live-card">
    <div class="flex flex-col gap-4">
      <Card>
        <div class="flex justify-between items-center gap-4">
          <span>Presets:</span>
          <div class="flex flex-wrap gap-2">
            <button
              v-for="preset in presets"
              :key="preset.name"
              class="px-3 py-1 rounded bg-gradient-to-r from-teal-600 to-cyan-500 text-white text-sm"
              @click="setPreset(preset)"
            >
              {{ preset.name }}
            </button>
          </div>
        </div>
      </Card>
      <div class="controls">
        <h3>Simulação Monte Carlo ao vivo</h3>
        <p>
          Política de estoque <strong>(s, Q)</strong> com demanda Poisson e lead time Triangular.
        </p>

        <div class="grid">
          <label>Simulações <input v-model.number="numSimulations" type="number" min="100" step="100" /></label>
          <label>Semanas <input v-model.number="numWeeks" type="number" min="4" max="104" /></label>
          <label>Estoque inicial <input v-model.number="initialInventory" type="number" min="0" /></label>
          <label>Ponto de pedido s <input v-model.number="s" type="number" min="0" /></label>
          <label>Quantidade Q <input v-model.number="q" type="number" min="1" /></label>
          <label>Lambda demanda <input v-model.number="poissonLambda" type="number" min="0.1" step="0.1" /></label>
          <label>Custo estoque (R$) <input v-model.number="holdingCostPerUnit" type="number" min="0" step="10" /></label>
          <label>Custo falta (R$) <input v-model.number="stockoutCostPerUnit" type="number" min="0" step="50" /></label>
          <label>Custo pedido (R$) <input v-model.number="orderCostFixed" type="number" min="0" step="50" /></label>
          <label>Bins histograma <input v-model.number="histogramBins" type="number" min="10" max="80" /></label>
          <label>Seed <input v-model.number="seed" type="number" min="1" step="1" /></label>
        </div>

        <button :disabled="isRunning" class="run-btn" @click="runSimulation">
          {{ isRunning ? 'Rodando...' : 'Rodar simulação' }}
        </button>

        <div class="progress-wrap" v-if="isRunning || completedSimulations > 0">
          <div class="progress-bar" :style="{ width: `${progress}%` }"></div>
        </div>
        <small v-if="isRunning || completedSimulations > 0">
          {{ completedSimulations.toLocaleString('pt-BR') }} / {{ numSimulations.toLocaleString('pt-BR') }} simulações
        </small>

        <div class="kpis" v-if="totalCosts.length > 0">
          <div><span>Custo médio anual:</span> <strong>{{ currency(meanCost) }}</strong></div>
          <div><span>Mediana do custo:</span> <strong>{{ currency(medianCost) }}</strong></div>
          <div><span>P95 de custo:</span> <strong>{{ currency(p95Cost) }}</strong></div>
          <div><span>Nível de serviço médio:</span> <strong>{{ meanServiceLevel.toFixed(2) }}%</strong></div>
        </div>
      </div>
    </div>

    <div class="charts">
      <div class="panel">
        <h4>Distribuição dos custos anuais</h4>
        <div class="canvas-wrap"><Bar :data="histogramData" :options="histogramOptions" /></div>
      </div>
      <div class="panel">
        <h4>Convergência do custo médio</h4>
        <div class="canvas-wrap"><Line :data="runningMeanData" :options="lineOptions" /></div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.live-card {
  display: grid;
  grid-template-columns: minmax(320px, 400px) minmax(0, 1fr);
  gap: 14px;
  width: 100%;
  max-width: 100%;
  max-height: 60vh;
  overflow-y: auto;
  overflow-x: hidden;
  padding-right: 4px;
}

.controls {
  border: 1px solid #d1d5db;
  border-radius: 14px;
  padding: 14px;
  background: linear-gradient(150deg, #f8fafc 0%, #ecfeff 100%);
  min-width: 0;
}

.controls h3 {
  margin: 0 0 6px;
  font-size: 1.1rem;
}

.controls p {
  margin: 0 0 12px;
  font-size: 0.9rem;
  line-height: 1.3;
}

.grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 8px;
}

label {
  display: flex;
  flex-direction: column;
  gap: 4px;
  font-size: 0.75rem;
}

input {
  border: 1px solid #cbd5e1;
  border-radius: 8px;
  padding: 5px 7px;
  font-size: 0.85rem;
  background: #fff;
}

.run-btn {
  margin-top: 12px;
  width: 100%;
  border: none;
  border-radius: 10px;
  padding: 8px 10px;
  font-weight: 600;
  color: #fff;
  background: linear-gradient(90deg, #0f766e 0%, #0891b2 100%);
  cursor: pointer;
}

.run-btn:disabled {
  opacity: 0.7;
  cursor: not-allowed;
}

.progress-wrap {
  margin-top: 10px;
  width: 100%;
  height: 10px;
  border-radius: 999px;
  overflow: hidden;
  background: #cbd5e1;
}

.progress-bar {
  height: 100%;
  background: linear-gradient(90deg, #f97316 0%, #ea580c 100%);
  transition: width 120ms linear;
}

.kpis {
  margin-top: 10px;
  display: grid;
  gap: 4px;
  font-size: 0.82rem;
}

.kpis div {
  display: flex;
  justify-content: space-between;
  gap: 8px;
}

.charts {
  display: grid;
  grid-template-rows: 1fr 1fr;
  gap: 12px;
  min-width: 0;
}

.panel {
  border: 1px solid #d1d5db;
  border-radius: 14px;
  padding: 10px 12px;
  background: #fff;
  min-width: 0;
}

.panel h4 {
  margin: 0 0 8px;
  font-size: 0.95rem;
}

.canvas-wrap {
  height: 195px;
  width: 100%;
}

@media (max-width: 1100px) {
  .live-card {
    grid-template-columns: 1fr;
    max-height: 66vh;
  }

  .charts {
    grid-template-rows: auto;
    grid-template-columns: 1fr;
  }

  .canvas-wrap {
    height: 220px;
  }
}
</style>
