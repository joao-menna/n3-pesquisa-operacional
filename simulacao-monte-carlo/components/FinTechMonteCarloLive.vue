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

type ScenarioPreset = {
	name: string
	numSimulations: number
	histogramBins: number
	seed: number
	phaseA: { min: number; mode: number; max: number }
	phaseB: { min: number; mode: number; max: number }
	phaseC: { min: number; mode: number; max: number }
	phaseBPertLambda: number
	risk1Probability: number
	risk1ImpactMin: number
	risk1ImpactMax: number
	risk2Probability: number
	risk2Impact: number
}

const presets: ScenarioPreset[] = [
	{
		name: 'Base',
		numSimulations: 10_000,
		histogramBins: 40,
		seed: 42,
		phaseA: { min: 20, mode: 30, max: 45 },
		phaseB: { min: 25, mode: 35, max: 55 },
		phaseC: { min: 10, mode: 15, max: 30 },
		phaseBPertLambda: 4,
		risk1Probability: 0.3,
		risk1ImpactMin: 10,
		risk1ImpactMax: 20,
		risk2Probability: 0.15,
		risk2Impact: 12,
	},
	{
		name: 'Mais folga',
		numSimulations: 10_000,
		histogramBins: 40,
		seed: 42,
		phaseA: { min: 18, mode: 27, max: 40 },
		phaseB: { min: 23, mode: 33, max: 50 },
		phaseC: { min: 9, mode: 13, max: 26 },
		phaseBPertLambda: 4,
		risk1Probability: 0.22,
		risk1ImpactMin: 8,
		risk1ImpactMax: 16,
		risk2Probability: 0.1,
		risk2Impact: 10,
	},
	{
		name: 'Crítico',
		numSimulations: 10_000,
		histogramBins: 40,
		seed: 42,
		phaseA: { min: 22, mode: 33, max: 48 },
		phaseB: { min: 28, mode: 39, max: 60 },
		phaseC: { min: 11, mode: 17, max: 34 },
		phaseBPertLambda: 4,
		risk1Probability: 0.38,
		risk1ImpactMin: 12,
		risk1ImpactMax: 24,
		risk2Probability: 0.22,
		risk2Impact: 14,
	},
]

const deadlineDays = 90

const numSimulations = ref(10_000)
const histogramBins = ref(40)
const seed = ref(42)
const activePreset = ref('Base')

const phaseA = ref({ min: 20, mode: 30, max: 45 })
const phaseB = ref({ min: 25, mode: 35, max: 55 })
const phaseC = ref({ min: 10, mode: 15, max: 30 })
const phaseBPertLambda = ref(4)
const risk1Probability = ref(0.3)
const risk1ImpactMin = ref(10)
const risk1ImpactMax = ref(20)
const risk2Probability = ref(0.15)
const risk2Impact = ref(12)

const isRunning = ref(false)
const progress = ref(0)
const completedSimulations = ref(0)

const totalDurations = ref<number[]>([])
const runningMeans = ref<number[]>([])
const phaseADurations = ref<number[]>([])
const phaseBDurations = ref<number[]>([])
const phaseCDurations = ref<number[]>([])
const phaseGroupDurations = ref<number[]>([])
const riskGroupDurations = ref<number[]>([])
const risk1Impacts = ref<number[]>([])
const risk2Impacts = ref<number[]>([])

function mulberry32(initialSeed: number) {
	let t = initialSeed >>> 0

	return () => {
		t += 0x6D2B79F5
		let x = Math.imul(t ^ (t >>> 15), t | 1)
		x ^= x + Math.imul(x ^ (x >>> 7), x | 61)
		return ((x ^ (x >>> 14)) >>> 0) / 4294967296
	}
}

function sampleUniform(min: number, max: number, random: () => number) {
	return min + (max - min) * random()
}

function sampleTriangular(min: number, mode: number, max: number, random: () => number) {
	const u = random()
	const c = (mode - min) / (max - min)

	if (u <= c) {
		return min + Math.sqrt(u * (max - min) * (mode - min))
	}

	return max - Math.sqrt((1 - u) * (max - min) * (max - mode))
}

function sampleStandardNormal(random: () => number) {
	const u1 = Math.max(Number.EPSILON, random())
	const u2 = random()
	return Math.sqrt(-2 * Math.log(u1)) * Math.cos(2 * Math.PI * u2)
}

function sampleGamma(shape: number, random: () => number): number {
	if (shape < 1) {
		const boost = sampleGamma(shape + 1, random)
		return boost * Math.pow(Math.max(Number.EPSILON, random()), 1 / shape)
	}

	const d = shape - 1 / 3
	const c = 1 / Math.sqrt(9 * d)

	while (true) {
		let x = sampleStandardNormal(random)
		let v = 1 + c * x

		while (v <= 0) {
			x = sampleStandardNormal(random)
			v = 1 + c * x
		}

		v **= 3
		const u = random()

		if (u < 1 - 0.0331 * x ** 4) {
			return d * v
		}

		if (Math.log(u) < 0.5 * x * x + d * (1 - v + Math.log(v))) {
			return d * v
		}
	}
}

function sampleBeta(alpha: number, beta: number, random: () => number) {
	const x = sampleGamma(alpha, random)
	const y = sampleGamma(beta, random)
	return x / (x + y)
}

function samplePERT(min: number, mode: number, max: number, lambda: number, random: () => number) {
	const alpha = 1 + lambda * ((mode - min) / (max - min))
	const beta = 1 + lambda * ((max - mode) / (max - min))
	return min + sampleBeta(alpha, beta, random) * (max - min)
}

function correlation(valuesA: number[], valuesB: number[]) {
	if (valuesA.length === 0 || valuesA.length !== valuesB.length) return 0

	const meanA = valuesA.reduce((acc, value) => acc + value, 0) / valuesA.length
	const meanB = valuesB.reduce((acc, value) => acc + value, 0) / valuesB.length

	let covariance = 0
	let varianceA = 0
	let varianceB = 0

	for (let index = 0; index < valuesA.length; index += 1) {
		const deltaA = valuesA[index] - meanA
		const deltaB = valuesB[index] - meanB
		covariance += deltaA * deltaB
		varianceA += deltaA * deltaA
		varianceB += deltaB * deltaB
	}

	if (varianceA === 0 || varianceB === 0) return 0

	return covariance / Math.sqrt(varianceA * varianceB)
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

function formatDays(value: number) {
	return `${value.toLocaleString('pt-BR', { maximumFractionDigits: 1 })} dias`
}

function formatPercent(value: number) {
	return `${(value * 100).toLocaleString('pt-BR', { maximumFractionDigits: 0 })}%`
}

function simulateBatch(
	batchSize: number,
	random: () => number,
	currentTotals: number[],
	currentRunningMean: number[],
	currentPhaseA: number[],
	currentPhaseB: number[],
	currentPhaseC: number[],
	currentPhaseGroup: number[],
	currentRiskGroup: number[],
	currentRisk1: number[],
	currentRisk2: number[],
) {
	for (let simulation = 0; simulation < batchSize; simulation += 1) {
		const durationA = sampleTriangular(phaseA.value.min, phaseA.value.mode, phaseA.value.max, random)
		const durationB = samplePERT(
			phaseB.value.min,
			phaseB.value.mode,
			phaseB.value.max,
			phaseBPertLambda.value,
			random,
		)
		const durationC = sampleTriangular(phaseC.value.min, phaseC.value.mode, phaseC.value.max, random)

		const risk1 = random() < risk1Probability.value
			? sampleUniform(risk1ImpactMin.value, risk1ImpactMax.value, random)
			: 0

		const risk2 = random() < risk2Probability.value
			? risk2Impact.value
			: 0

		const phaseTotal = durationA + durationB + durationC
		const riskTotal = risk1 + risk2
		const totalDuration = phaseTotal + riskTotal

		currentPhaseA.push(durationA)
		currentPhaseB.push(durationB)
		currentPhaseC.push(durationC)
		currentPhaseGroup.push(phaseTotal)
		currentRiskGroup.push(riskTotal)
		currentRisk1.push(risk1)
		currentRisk2.push(risk2)
		currentTotals.push(totalDuration)

		const sampleCount = currentTotals.length
		const previousMean = sampleCount > 1 ? currentRunningMean[sampleCount - 2] : 0
		const nextMean = previousMean + (totalDuration - previousMean) / sampleCount
		currentRunningMean.push(nextMean)
	}
}

async function runSimulation() {
	if (isRunning.value) return

	isRunning.value = true
	progress.value = 0
	completedSimulations.value = 0

	if (seed.value === 0) {
		seed.value = Math.floor(Math.random() * 0xFFFFFFFF) + 1
	}

	const random = mulberry32(seed.value)
	const currentTotals: number[] = []
	const currentRunningMean: number[] = []
	const currentPhaseA: number[] = []
	const currentPhaseB: number[] = []
	const currentPhaseC: number[] = []
	const currentPhaseGroup: number[] = []
	const currentRiskGroup: number[] = []
	const currentRisk1: number[] = []
	const currentRisk2: number[] = []

	const chunkSize = Math.max(100, Math.floor(numSimulations.value / 40))

	while (currentTotals.length < numSimulations.value) {
		const remaining = numSimulations.value - currentTotals.length
		const batchSize = Math.min(chunkSize, remaining)

		simulateBatch(
			batchSize,
			random,
			currentTotals,
			currentRunningMean,
			currentPhaseA,
			currentPhaseB,
			currentPhaseC,
			currentPhaseGroup,
			currentRiskGroup,
			currentRisk1,
			currentRisk2,
		)

		completedSimulations.value = currentTotals.length
		progress.value = (currentTotals.length / numSimulations.value) * 100

		totalDurations.value = [...currentTotals]
		runningMeans.value = [...currentRunningMean]
		phaseADurations.value = [...currentPhaseA]
		phaseBDurations.value = [...currentPhaseB]
		phaseCDurations.value = [...currentPhaseC]
		phaseGroupDurations.value = [...currentPhaseGroup]
		riskGroupDurations.value = [...currentRiskGroup]
		risk1Impacts.value = [...currentRisk1]
		risk2Impacts.value = [...currentRisk2]

		await new Promise((resolve) => setTimeout(resolve, 0))
	}

	isRunning.value = false
}

const meanDuration = computed(() => {
	if (totalDurations.value.length === 0) return 0
	const sum = totalDurations.value.reduce((acc, value) => acc + value, 0)
	return sum / totalDurations.value.length
})

const medianDuration = computed(() => percentile(totalDurations.value, 50))
const p95Duration = computed(() => percentile(totalDurations.value, 95))

const successProbability = computed(() => {
	if (totalDurations.value.length === 0) return 0
	const onTime = totalDurations.value.filter((duration) => duration <= deadlineDays).length
	return (onTime / totalDurations.value.length) * 100
})

const meanDelay = computed(() => {
	if (totalDurations.value.length === 0) return 0
	const delayed = totalDurations.value.map((duration) => Math.max(0, duration - deadlineDays))
	return delayed.reduce((acc, value) => acc + value, 0) / delayed.length
})

const contingencyDays = computed(() => Math.max(0, p95Duration.value - deadlineDays))

const phaseMacroCorrelation = computed(() => correlation(phaseGroupDurations.value, totalDurations.value))
const riskMacroCorrelation = computed(() => correlation(riskGroupDurations.value, totalDurations.value))

const dominantMacro = computed(() => {
	if (totalDurations.value.length === 0) return 'aguardando simulação'
	return phaseMacroCorrelation.value >= riskMacroCorrelation.value
		? 'Duração das fases'
		: 'Eventos de risco'
})

const factorSensitivityData = computed(() => {
	const factors = [
		{ label: 'Fase A', value: Math.abs(correlation(phaseADurations.value, totalDurations.value)) },
		{ label: 'Fase B', value: Math.abs(correlation(phaseBDurations.value, totalDurations.value)) },
		{ label: 'Fase C', value: Math.abs(correlation(phaseCDurations.value, totalDurations.value)) },
		{ label: 'Risco 1', value: Math.abs(correlation(risk1Impacts.value, totalDurations.value)) },
		{ label: 'Risco 2', value: Math.abs(correlation(risk2Impacts.value, totalDurations.value)) },
	]

	return {
		labels: factors.map((factor) => factor.label),
		datasets: [
			{
				label: 'Correlação com a duração total',
				data: factors.map((factor) => factor.value),
				backgroundColor: [
					'rgba(14, 165, 233, 0.75)',
					'rgba(59, 130, 246, 0.75)',
					'rgba(99, 102, 241, 0.75)',
					'rgba(249, 115, 22, 0.75)',
					'rgba(234, 88, 12, 0.75)',
				],
				borderColor: [
					'rgba(14, 165, 233, 1)',
					'rgba(59, 130, 246, 1)',
					'rgba(99, 102, 241, 1)',
					'rgba(249, 115, 22, 1)',
					'rgba(234, 88, 12, 1)',
				],
				borderWidth: 1,
			},
		],
	}
})

const histogramData = computed(() => {
	if (totalDurations.value.length === 0) {
		return {
			labels: [],
			datasets: [
				{
					label: 'Frequência',
					data: [],
					backgroundColor: 'rgba(14, 165, 233, 0.7)',
					borderColor: 'rgba(59, 130, 246, 1)',
					borderWidth: 1,
				},
			],
		}
	}

	const minDuration = Math.min(...totalDurations.value)
	const maxDuration = Math.max(...totalDurations.value)
	const binCount = Math.max(10, histogramBins.value)
	const binWidth = Math.max(1, (maxDuration - minDuration) / binCount)
	const counts = new Array(binCount).fill(0)

	for (const duration of totalDurations.value) {
		const rawIndex = Math.floor((duration - minDuration) / binWidth)
		const index = Math.min(binCount - 1, Math.max(0, rawIndex))
		counts[index] += 1
	}

	const labels = counts.map((_, index) => {
		const start = minDuration + index * binWidth
		const end = start + binWidth
		return `${Math.round(start)}-${Math.round(end)} dias`
	})

	return {
		labels,
		datasets: [
			{
				label: 'Distribuição da duração total',
				data: counts,
				backgroundColor: 'rgba(14, 165, 233, 0.7)',
				borderColor: 'rgba(59, 130, 246, 1)',
				borderWidth: 1,
			},
		],
	}
})

const runningMeanData = computed(() => {
	const step = Math.max(1, Math.floor(runningMeans.value.length / 120))
	const compactMeans: number[] = []
	const labels: string[] = []

	for (let index = 0; index < runningMeans.value.length; index += step) {
		compactMeans.push(runningMeans.value[index])
		labels.push(String(index + 1))
	}

	return {
		labels,
		datasets: [
			{
				label: 'Média acumulada da duração (dias)',
				data: compactMeans,
				borderColor: 'rgba(249, 115, 22, 1)',
				backgroundColor: 'rgba(249, 115, 22, 0.2)',
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
				text: 'Simulações processadas',
			},
		},
		y: {
			title: {
				display: true,
				text: 'Dias',
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
				text: 'Faixa de duração total',
			},
		},
		y: {
			title: {
				display: true,
				text: 'Frequência',
			},
		},
	},
}

const sensitivityOptions = {
	...chartOptions,
	scales: {
		x: {
			ticks: {
				maxRotation: 0,
			},
		},
		y: {
			beginAtZero: true,
			suggestedMax: 1,
			title: {
				display: true,
				text: 'Correlação de Pearson',
			},
		},
	},
}

function setPreset(preset: ScenarioPreset) {
	activePreset.value = preset.name
	numSimulations.value = preset.numSimulations
	histogramBins.value = preset.histogramBins
	seed.value = preset.seed
	phaseA.value = { ...preset.phaseA }
	phaseB.value = { ...preset.phaseB }
	phaseC.value = { ...preset.phaseC }
	phaseBPertLambda.value = preset.phaseBPertLambda
	risk1Probability.value = preset.risk1Probability
	risk1ImpactMin.value = preset.risk1ImpactMin
	risk1ImpactMax.value = preset.risk1ImpactMax
	risk2Probability.value = preset.risk2Probability
	risk2Impact.value = preset.risk2Impact
}
</script>

<template>
	<div class="live-card">
		<div class="flex flex-col gap-4 h-full">
			<Card>
				<div class="flex justify-between items-center gap-4">
					<span>Presets:</span>
					<div class="flex flex-wrap gap-2">
						<button
							v-for="preset in presets"
							:key="preset.name"
							:class="[
								'px-3 py-1 rounded text-sm transition-colors',
								preset.name === activePreset
									? 'bg-gradient-to-r from-sky-600 to-cyan-500 text-white shadow-md'
									: 'bg-slate-100 text-slate-700 hover:bg-slate-200',
							]"
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
					Projeto de pagamentos internacionais com prazo fixo de <strong>90 dias</strong>, fases sequenciais e dois riscos independentes.
				</p>

				<div class="preset-feedback">
					<div class="preset-feedback__title">
						<span>Preset ativo</span>
						<strong>{{ activePreset }}</strong>
					</div>
					<div class="preset-feedback__grid">
						<div><span>Simulações</span><strong>{{ numSimulations.toLocaleString('pt-BR') }}</strong></div>
						<div><span>Bins</span><strong>{{ histogramBins }}</strong></div>
						<div><span>Seed</span><strong>{{ seed }}</strong></div>
						<div><span>Lambda PERT</span><strong>{{ phaseBPertLambda.toFixed(0) }}</strong></div>
						<div><span>Fase A</span><strong>{{ phaseA.min }} / {{ phaseA.mode }} / {{ phaseA.max }} dias</strong></div>
						<div><span>Fase B</span><strong>{{ phaseB.min }} / {{ phaseB.mode }} / {{ phaseB.max }} dias</strong></div>
						<div><span>Fase C</span><strong>{{ phaseC.min }} / {{ phaseC.mode }} / {{ phaseC.max }} dias</strong></div>
						<div><span>Risco 1</span><strong>{{ formatPercent(risk1Probability) }} · {{ risk1ImpactMin }} a {{ risk1ImpactMax }} dias</strong></div>
						<div><span>Risco 2</span><strong>{{ formatPercent(risk2Probability) }} · {{ risk2Impact }} dias</strong></div>
					</div>
				</div>

				<div class="assumptions">
					<div>
						<span>Fase A:</span> <strong>Triangular 20 / 30 / 45</strong>
					</div>
					<div>
						<span>Fase B:</span> <strong>PERT 25 / 35 / 55</strong>
					</div>
					<div>
						<span>Fase C:</span> <strong>Triangular 10 / 15 / 30</strong>
					</div>
					<div>
						<span>Risco 1:</span> <strong>30% e impacto uniforme de 10 a 20 dias</strong>
					</div>
					<div>
						<span>Risco 2:</span> <strong>15% e impacto fixo de 12 dias</strong>
					</div>
				</div>

				<div class="grid">
					<label>Simulações <input v-model.number="numSimulations" type="number" min="100" step="100" /></label>
					<label>Bins do histograma <input v-model.number="histogramBins" type="number" min="10" max="80" /></label>
					<label>Seed <input v-model.number="seed" type="number" min="0" step="1" /></label>
				</div>

				<button :disabled="isRunning" class="run-btn" @click="runSimulation">
					{{ isRunning ? 'Rodando...' : 'Rodar simulação' }}
				</button>

				<div v-if="isRunning || completedSimulations > 0" class="progress-wrap">
					<div class="progress-bar" :style="{ width: `${progress}%` }"></div>
				</div>

				<small v-if="isRunning || completedSimulations > 0">
					{{ completedSimulations.toLocaleString('pt-BR') }} / {{ numSimulations.toLocaleString('pt-BR') }} simulações
				</small>

				<div v-if="totalDurations.length > 0" class="kpis">
					<div><span>Probabilidade de cumprir 90 dias:</span> <strong>{{ successProbability.toFixed(2) }}%</strong></div>
					<div><span>Duração média:</span> <strong>{{ formatDays(meanDuration) }}</strong></div>
					<div><span>Mediana da duração:</span> <strong>{{ formatDays(medianDuration) }}</strong></div>
					<div><span>Prazo para 95% de certeza:</span> <strong>{{ formatDays(p95Duration) }}</strong></div>
					<div><span>Folga recomendada:</span> <strong>{{ formatDays(contingencyDays) }}</strong></div>
					<div><span>Atraso médio esperado:</span> <strong>{{ formatDays(meanDelay) }}</strong></div>
				</div>

				<div v-if="totalDurations.length > 0" class="macro-summary">
					<span>Análise macro</span>
					<strong>{{ dominantMacro }}</strong>
					<small>
						Fases: {{ phaseMacroCorrelation.toFixed(3) }} · Riscos: {{ riskMacroCorrelation.toFixed(3) }}
					</small>
				</div>
			</div>
		</div>

		<div class="charts">
			<div class="panel">
				<h4>Distribuição das durações totais</h4>
				<div class="canvas-wrap">
					<Bar :data="histogramData" :options="histogramOptions" />
				</div>
			</div>

			<div class="panel">
				<h4>Convergência da duração média</h4>
				<div class="canvas-wrap">
					<Line :data="runningMeanData" :options="lineOptions" />
				</div>
			</div>

			<div class="panel">
				<h4>Sensibilidade por fator</h4>
				<div class="canvas-wrap">
					<Bar :data="factorSensitivityData" :options="sensitivityOptions" />
				</div>
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
	max-height: 100%;
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

.preset-feedback {
	display: grid;
	gap: 8px;
	margin-bottom: 12px;
	padding: 12px;
	border-radius: 12px;
	border: 1px solid #bae6fd;
	background: linear-gradient(145deg, rgba(255, 255, 255, 0.88), rgba(224, 242, 254, 0.78));
	font-size: 0.78rem;
}

.preset-feedback__title {
	display: flex;
	justify-content: space-between;
	gap: 10px;
	align-items: center;
	font-size: 0.82rem;
	color: #0f172a;
}

.preset-feedback__title span {
	color: #0369a1;
	text-transform: uppercase;
	letter-spacing: 0.04em;
	font-size: 0.72rem;
}

.preset-feedback__grid {
	display: grid;
	grid-template-columns: repeat(2, minmax(0, 1fr));
	gap: 8px 12px;
}

.preset-feedback__grid > div {
	display: flex;
	flex-direction: column;
	gap: 2px;
	padding: 8px 10px;
	border-radius: 10px;
	background: rgba(255, 255, 255, 0.75);
	border: 1px solid rgba(125, 211, 252, 0.4);
}

.preset-feedback__grid span {
	color: #475569;
	font-size: 0.7rem;
	text-transform: uppercase;
	letter-spacing: 0.04em;
}

.preset-feedback__grid strong {
	font-size: 0.82rem;
	color: #0f172a;
}

.assumptions {
	display: grid;
	gap: 6px;
	margin-bottom: 12px;
	padding: 10px 12px;
	border-radius: 12px;
	background: rgba(255, 255, 255, 0.75);
	border: 1px solid #dbeafe;
	font-size: 0.78rem;
}

.assumptions > div {
	display: flex;
	justify-content: space-between;
	gap: 10px;
}

.assumptions span {
	color: #334155;
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

.macro-summary {
	margin-top: 10px;
	display: grid;
	gap: 2px;
	padding: 10px 12px;
	border-radius: 12px;
	background: linear-gradient(145deg, rgba(15, 23, 42, 0.04), rgba(14, 165, 233, 0.08));
	border: 1px solid rgba(14, 165, 233, 0.18);
	font-size: 0.82rem;
}

.macro-summary span {
	color: #475569;
}

.charts {
	display: grid;
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
	max-height: 100%;
}

.canvas-wrap {
	height: 260px;
	width: 100%;
}
</style>
