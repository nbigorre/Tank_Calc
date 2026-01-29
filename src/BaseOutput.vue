<script setup>
import { ref, watch } from 'vue'

import {
	Chart as ChartJS,
	CategoryScale,
	LinearScale,
	PointElement,
	LineElement,
	Title,
	Tooltip,
	Legend,
	scales
} from 'chart.js'
import { Line } from 'vue-chartjs';

import zoomPlugin from 'chartjs-plugin-zoom';
ChartJS.register(
	CategoryScale,
	LinearScale,
	PointElement,
	LineElement,
	Title,
	Tooltip,
	Legend,
	zoomPlugin
)

const chart_option = {
	plugins: {
		zoom: {
			zoom: {
				wheel: {
					enabled: true,
				},
				pinch: {
					enabled: true
				},
				mode: 'x',
			}
		}
	},
	onUpdate: (chart) => {
		chart.resetZoom()
	}
}
const pred_chart_option = {
	plugins: {
		tooltip: {
			enabled: true
		},
		tooltip: {
			callbacks: {
				title: (items) => {
					const x = items[0].parsed.x
					return `Volume: ${parseFloat(x.toFixed(1))} m³`
				},
				label: (item) => {
					const y = item.parsed.y
					return `Satisfaction: ${y}%`
				}
			}
		}
	},
	interaction: {
		mode: 'nearest',
		intersect: false
	},
	scales: {
		x: {
			type: "linear",
			ticks: {
				callback: value => `${value} m³`
			},
			title: {
				display: true,
				text: 'Volume nécessaire (m³)'
			}
		},
		y: {
			ticks: {
				callback: value => `${value}%`
			},
			title: {
				display: true,
				text: 'Satisfaction du besoin (%)'
			}
		}
	},
	onUpdate: (chart) => {
		chart.resetZoom();
	}
}


const props = defineProps({
	table_data: {
		type: Object,
		default: () => ({ stats_year: new Map(), min: {}, max: {}, mean: {} })
	},
	tank_data: {
		type: Object,
		default: () => ({
			labels: [],
			datasets: [
				{
					label: "Volume d'eau stocké dans la cuve (m3)",
					borderColor: "#000FAF",
					borderWidth: 1,
					backgroundColor: "#000FAF",
					pointRadius: 0,
					pointHitRadius: 10,
					data: []
				}
			]
		})
	},
	prediction_data: {
		type: Object,
		default: () => ({
			labels: [],
			datasets: [
				{
					label: "Volume de cuve necessaire pour Satisfaire le besoin",
					borderColor: "#000FAF",
					borderWidth: 1,
					backgroundColor: "#000FAF",
					pointRadius: 0,
					pointHitRadius: 10,
					data: []
				}
			]
		})
	}
});


</script>

<template>
	<div class="output-section">
		<!-- Table with 4 columns and 20 rows -->
		<div class="table-container">
			<table>
				<thead>
					<tr>
						<th>Année</th>
						<th>Nombre de jours avec la cuve vide</th>
						<th>Volume economisé (m<sup>3</sup>)</th>
						<th>Besoin complémentaire (m<sup>3</sup>)</th>
						<th>Satisfaction de Besoin</th>
					</tr>
				</thead>
				<tbody>
					<tr v-for="(value, key) of table_data.stats_year" :key="key" :id="key">
						<td id="year">{{ value[0] }}</td>
						<td id="empty">{{ value[1].empty }}</td>
						<td id="saved">{{ value[1].saved }}</td>
						<td id="needed">{{ value[1].needed }}</td>
						<td id="percent">{{ value[1].percent }}</td>
					</tr>
					<tr v-for="(value, key) of [['Minimum', table_data.min], ['Maximum', table_data.max], ['Moyenne', table_data.mean]]"
						:key="value[0]" :id="value[0]">
						<td id="global_stat">{{ value[0] }}</td>
						<td id="empty">{{ value[1].empty }}</td>
						<td id="saved">{{ value[1].saved }}</td>
						<td id="needed">{{ value[1].needed }}</td>
						<td id="percent">{{ value[1].percent }}</td>
					</tr>
					<tr></tr>
				</tbody>
			</table>
			<i>Calcul réalisé à partir des pluies journalières (Source : Météo France, modèle SAFRAN) données issues de
				l’API GEOSAS)</i>
		</div>
		<div class="chart-container">
			<Line :data="tank_data" :options="chart_option" class="chart" />
			<Line :data="prediction_data" :options="pred_chart_option" class="chart" />
		</div>
	</div>
</template>

<style>
.output-section {
	display: grid;
	grid-template-columns: 40% 60%;

	padding: 20px;
	border: 1px solid #ccc;
}

.table-container {
	border: 1px solid #ddd;
	padding: 10px;
}

.chart {
	padding: 20px;
	border: 1px solid #ccc;
}


table {
	width: 100%;
	border-collapse: collapse;
  	table-layout: fixed;
}

table th,
table td {
	font-size: clamp(1px, 0.9vw, 14px);
  	padding: 6px;
	text-align: center;
	border: 1px solid #ddd;
}

table th {
	background-color: #f4f4f4;
}

#Minimum,
#Maximum,
#Moyenne {
	background-color: #E4E4E4;
	font-weight: bold;
}
</style>