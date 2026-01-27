<script setup>
import { computed, ref } from 'vue';
import BaseInput from './BaseInput.vue';
import BaseOutput from './BaseOutput.vue';

let currentLatitude = null;
let currentLongitude = null;

let rainData = null;

const tank_data = ref({ labels: [], datasets: [{ label: 'Volume d\'eau stocké dans la cuve (m3)', borderColor: "#000FAF", borderWidth: 1, backgroundColor: "#000FAF", pointRadius: 0, pointHitRadius: 10, data: [] }] });
const table_data = ref({ stats_year: new Map(), min: {}, max: {}, mean: {} })
const prediction_data = ref({ labels: [], datasets: [{ label: 'Volume de cuve necessaire pour Satisfaire le besoin', borderColor: "#000FAF", borderWidth: 1, backgroundColor: "#000FAF", pointRadius: 0, pointHitRadius: 10, data: [] }] })

function retrieveRainData(lat, long, start_year, stop_year) {
	const url = `https://api.geosas.fr/edr/collections/safran-isba/position?coords=POINT(${long} ${lat})&datetime=${start_year}-01-01/${stop_year}-12-31&parameter-name=PRENEI_Q,PRELIQ_Q&crs=EPSG:4326&f=CoverageJSON`
	const xmlHttp = new XMLHttpRequest();
	xmlHttp.open("GET", url, false);
	xmlHttp.send(null);
	const response = JSON.parse(xmlHttp.responseText);
	if (response.response != undefined) {
		window.alert("Erreur lors de la requête.");
		return false;
	}
	rainData = Array(response.domain.axes.t.values.length)

	for (let i = 0; i < response.domain.axes.t.values.length; i++) {
		let date = response.domain.axes.t.values[i].slice(0, 10);
		let rain = response.ranges.PRELIQ_Q.values[i] + response.ranges.PRENEI_Q.values[i];
		rainData[i] = { date, rain };
	}
	return true;
}

function computeTank(rainData, surface, ratio, is_full, monthly, volume = Number.MAX_SAFE_INTEGER) {
	const tank = Array(rainData.length);
	const consumed = Array(rainData.length);
	const needed = Array(rainData.length);
	[tank[0], consumed[0], needed[0]] = [(is_full ? volume : 0), 0, 0]

	for (let i = 1; i < rainData.length; i++) {
		const rain = rainData[i].rain;
		const daily_cons = monthly[parseInt(rainData[i].date.slice(5, 7), 10) - 1];
		const volume_increased = Math.min(volume, Math.max(0, tank[i - 1] + (rain < 2 ? 0 : rain) * surface * ratio / 1000));
		tank[i] = Math.max(0, volume_increased - daily_cons);
		consumed[i] = Math.min(volume_increased, daily_cons);
		needed[i] = Math.max(0, daily_cons - consumed[i]);
	}
	return { tank, consumed, needed };
}

function getEfficiency(computed) {
	const consumed_sum = computed.consumed.reduce((acc, v) => acc + v, 0);
	const needed_sum = computed.needed.reduce((acc, v) => acc + v, 0);
	return parseInt(consumed_sum / (needed_sum + consumed_sum) * 100);
}

function recursivePredict(efficiency_curve, minimum, maximum, func) {
	if (minimum + 1 >= maximum) {
		return;
	}
	const mid_volume = (efficiency_curve[maximum].min + efficiency_curve[minimum].max) / 2;
	const mid_efficiency = func(mid_volume);
	if (efficiency_curve[mid_efficiency] === undefined) {
		efficiency_curve[mid_efficiency] = { min: mid_volume, max: mid_volume };
	} else {
		efficiency_curve[mid_efficiency].min = Math.min(efficiency_curve[mid_efficiency].min, mid_volume);
		efficiency_curve[mid_efficiency].max = Math.max(efficiency_curve[mid_efficiency].max, mid_volume);
	}
	recursivePredict(efficiency_curve, minimum, mid_efficiency, func);
	recursivePredict(efficiency_curve, mid_efficiency, maximum, func);
}

function computePrediction(rainData, surface, ratio, monthly) {
	const computed = computeTank(rainData, surface, ratio, false, monthly);
	const max_volume = Math.max(...computed.tank.map((item, i) => item + computed.consumed[i]));
	const max_efficiency = getEfficiency(computed);
	const func = (volume) => getEfficiency(computeTank(rainData, surface, ratio, false, monthly, volume));
	const efficiency_curve = Array(max_efficiency+1);
	efficiency_curve[0] = { min: 0, max: 0 };
	efficiency_curve[max_efficiency] = { min: max_volume, max: max_volume };
	recursivePredict(efficiency_curve, 0, max_efficiency, func);
	return efficiency_curve;

}

function computeStatistics(date, consumed, needed) {
	const stats_year = new Map();
	for (let i = 0; i < date.length; i++) {
		const year = date[i].slice(0, 4);
		if (!stats_year.has(year)) {
			stats_year.set(year, { empty: 0, saved: 0, needed: 0, percent: 0 })
		}
		if (needed[i] > 0) {
			stats_year.get(year).needed += needed[i];
			stats_year.get(year).empty++;
		}
		stats_year.get(year).saved += consumed[i];
	}
	const min = { empty: 0, saved: 0, needed: 0, percent: 0 };
	const max = { empty: 0, saved: 0, needed: 0, percent: 0 };
	const mean = { empty: 0, saved: 0, needed: 0, percent: 0 };
	stats_year.forEach((v) => {
		v.percent = parseFloat((v.saved / (v.needed + v.saved) * 100).toFixed(1));
		v.empty = parseInt(v.empty);
		v.saved = parseInt(v.saved);
		v.needed = parseInt(v.needed);

		min.empty = Math.min(min.empty, v.empty);
		min.saved = Math.min(min.saved, v.saved);
		min.needed = Math.min(min.needed, v.needed);
		min.percent = Math.min(min.percent, v.percent);
		max.empty = Math.max(max.empty, v.empty);
		max.saved = Math.max(max.saved, v.saved);
		max.needed = Math.max(max.needed, v.needed);
		max.percent = Math.max(max.percent, v.percent);
		mean.empty += v.empty;
		mean.saved += v.saved;
		mean.needed += v.needed;
		mean.percent += v.percent;
	});
	mean.empty = parseInt(mean.empty / stats_year.size);
	mean.saved = parseInt(mean.saved / stats_year.size);
	mean.needed = parseInt(mean.needed / stats_year.size);
	mean.percent = parseFloat((mean.percent / stats_year.size).toFixed(1));
	return { stats_year, min, max, mean };
}

function computeSubmission(event) {
	if (event.lat != currentLatitude || event.long != currentLongitude) {
		if (!retrieveRainData(event.lat, event.long, event.start_year, event.stop_year))
			return;
		currentLatitude = event.lat;
		currentLongitude = event.long;
	}
	const dates = rainData.map((item) => item.date);
	const computed_tank = computeTank(rainData, event.surface, event.ratio, event.is_full, event.monthly, event.volume);
	const computed_stats = computeStatistics(dates, computed_tank.consumed, computed_tank.needed);
	const computed_preds = computePrediction(rainData, event.surface, event.ratio, event.monthly);
	table_data.value = computed_stats;
	tank_data.value = {
		labels: dates,
		datasets: [{ label: 'Volume d\'eau stocké dans la cuve (m3)', borderColor: "#000FAF", borderWidth: 1, backgroundColor: "#000FAF", pointRadius: 0, pointHitRadius: 10, data: computed_tank.tank }]
	}
	prediction_data.value = {//[...computed_preds.keys()]
		datasets: [{ label: 'Volume de cuve necessaire pour Satisfaire le besoin', borderColor: "#000FAF", borderWidth: 1, backgroundColor: "#000FAF", pointRadius: 0, pointHitRadius: 10, data: computed_preds.map((item, i) => {return {x: item.min, y: i};}) }]
	}
}

</script>

<template>
	<div class="container">
		<BaseInput @submit="computeSubmission" />
		<BaseOutput :tank_data="tank_data" :table_data="table_data" :prediction_data="prediction_data" />
	</div>
</template>

<style scoped>
.container {
	display: grid;
	grid-template-columns: 20% 80%;
}
</style>
