<script setup>

import { computed, ref, watch } from 'vue';
import data from './data.json';

const emit = defineEmits({
	submit({ lat, long, start_year, stop_year, volume, is_full, surface, ratio, monthly }) { return true; },
	exportEvent({ lat, long, start_year, stop_year, volume, is_full, surface, ratio, monthly, departement, commune }) { return true }
});

const selectedDepartement = ref('');
const selectedCommune = ref('');

var selectedLatitude = null;
var selectedLongitude = null;

var start_year = 2011
var stop_year = new Date().getFullYear() - 1

var volume = 0;
var is_full = true;
var surface = 0;
var ratio = 0.9;

const monthlyValues = Array(12).fill(1)

const uniqueDepartments = computed(() => {
	return [...(new Set([...data].map(item => item.departement)))].sort((a, b) => {
		return parseInt(a, 10) - parseInt(b, 10);
	});
});
const filteredCommunes = computed(() => {
	return [...data]
		.filter(item => item.departement === selectedDepartement.value)
		.map(item => item.commune)
		.sort((a, b) => a.localeCompare(b));
});

watch(selectedCommune, (newC) => {

	if (newC) {
		const selectedItem = data.find(item => item.commune === selectedCommune.value && item.departement === selectedDepartement.value);
		if (selectedItem) {
			selectedLatitude = selectedItem.latitude_mairie;
			selectedLongitude = selectedItem.longitude_mairie;
			return;
		}
	}
	selectedLatitude = null;
	selectedLongitude = null;
});

function validateData() {
	if (selectedLatitude === null || selectedLongitude === null) {
		window.alert('Veuillez selectionner une commune et un département.')
		return;
	}
	if (volume < 0) {
		window.alert('Le volume n\'est pas valide. ')
		return;
	}
	if (surface < 0) {
		window.alert('La surface n\'est pas valide.')
		return;
	}
	if (ratio < 0) {
		window.alert('Le taux de récupération n\'est pas valide.')
		return;
	}
	if ([...monthlyValues].filter(item => item < 0).length > 0) {
		windows.alert('Erreur dans les valeurs journalières.')
		return;
	}
}

function exportData() {
	validateData();
	emit('exportEvent', { lat: selectedLatitude, long: selectedLongitude, start_year, stop_year, volume, is_full, surface, ratio, monthly: monthlyValues, departement: selectedDepartement.value.slice(4), commune: selectedCommune.value.slice(4) });
}

function sendValues() {
	validateData();
	emit('submit', { lat: selectedLatitude, long: selectedLongitude, start_year, stop_year, volume, is_full, surface, ratio, monthly: monthlyValues });
}


</script>

<template>
	<div class="input-section">
		<form>
			<!-- First Select for Departements -->
			<select v-model="selectedDepartement">
				<option value="" disabled selected>Select a Departement</option>
				<option v-for="departement in uniqueDepartments" :key="departement" :value="departement">{{ departement
				}}
				</option>
			</select>

			<!-- Second Select for Communes based on selected Departement -->
			<select v-model="selectedCommune" :disabled="!selectedDepartement">
				<option value="" disabled selected>Select a Commune</option>
				<option v-for="commune in filteredCommunes" :key="commune" :value="commune">{{ commune }}</option>
			</select>

			<div>
				<label> Année de debut / fin</label><br>
				<input class="year-input" type="number" v-model.number="start_year" placeholder="Année de départ" />
				<input class="year-input" type="number" v-model.number="stop_year" placeholder="Année de fin" />
			</div>

			<!-- Number Inputs for Volume, Surface, and ratio -->
			<div>
				<label for="volume">Volume cuve (m<sup>3</sup>) : </label> <br>
				<input type="number" v-model.number="volume" id="volume" placeholder="Entrer le volume" min="0" /> <br>
				<label>
					Initialiser le calcul avec la cuve pleine :
					<input type="checkbox" v-model="is_full" />
				</label>
				<br>
				<br>
				<label for="surface">Surface de toiture collectée (m<sup>2</sup>) : </label> <br>
				<input type="number" v-model.number="surface" id="surface" placeholder="Entrer la surface" min="0" />
				<br>
				<div
					title="Taux de récupération de l'eau de toiture (dépend de la pente et de la nature du toit. Toit métallique : 0,9 ; toit en tuiles 0,8 ; Toit végétalisé : 0,4 ; terrasse gravillonnée : 0,55 ; Terrasse béton : 0,75). Le taux de récupération des pluies journalières inférieures à 2mm est fixé à zéro)">
					<label for="ratio">
						Taux de collecte <img src="./assets/info.png" width="16" height="16" /> :
					</label> <br>
					<input type="number" v-model.number="ratio" id="ratio" placeholder="Enter ratio" min="0" max="1"
						step="0.1" />
				</div>
			</div>

			<!-- Monthly Inputs -->
			<div class="monthly-inputs">
				<h3>Besoin en eau journalier (m<sup>3</sup>) en fonction du mois</h3>
				<div class="months-container">
					<div v-for="(value, index) in monthlyValues" :key="index" class="month-input">
						<label :for="'month-' + (index + 1)">
							{{ new Date(0, index).toLocaleString('default', { month: 'long' }) }}:
						</label>
						<input type="number" v-model.number="monthlyValues[index]" :id="'month-' + (index + 1)"
							:placeholder="'Month ' + (index + 1)" min="0" step="0.1" />
					</div>
				</div>
			</div>

			<!-- Actualiser Button -->
			<button type="button" @click="sendValues" class="actualiser-btn">Actualiser</button>
			<button type="button" @click="exportData" class="Export-btn">Exporter</button>
		</form>
	</div>
</template>

<style>
.input-section {
	padding: 20px;
	border: 1px solid #ccc;
	background-color: #FAFAFA;
}

.year-input {
	margin-bottom: 1px;
	padding: 1px;
	font-size: 1rem;
	width: 40%;
}

.monthly-inputs h3 {
	margin-top: 20px;
	margin-bottom: 15px;
}

.checkbox-row {
	display: flex;
	align-items: center;
	gap: 8px;
}

.months-container {
	display: grid;
	grid-template-columns: repeat(auto-fit, minmax(120px, 1fr));
}

.month-input {
	display: flex;
	flex-direction: column;
	align-items: center;
}

.month-input label {
	margin-bottom: 5px;
}

.month-input input {
	width: 80px;
	padding: 5px;
	text-align: center;
}

form {
	display: flex;
	flex-direction: column;
}

button {
	padding: 15px;
}

form select,
form input {
	margin-bottom: 15px;
	padding: 10px;
	font-size: 1rem;
	min-width: 0;
	width: 100%;
	box-sizing: border-box;
}

input[type="checkbox"] {
	width: auto;
	margin-bottom: 0;
}

h3,
label {
	font-weight: bold;
}
</style>