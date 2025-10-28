<script>
import data from './data.json';
import {
  Chart as ChartJS,
  CategoryScale,
  LinearScale,
  PointElement,
  LineElement,
  Title,
  Tooltip,
  Legend
} from 'chart.js'
import { Line } from 'vue-chartjs'

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

export default {
  components: {
    Line
  },
  data() {
    return {
      items: data,               // Original data containing both departments and communes
      selectedDepartement: '',    // Track the selected departement
      selectedCommune: '',       // Track the selected commune
      selectedLatitude: null,    // Track the latitude of the selected commune
      selectedLongitude: null,   // Track the longitude of the selected commune
      volume: null,              // Track the volume input
      surface: null,             // Track the surface input
      ratio: 0.9,          // Track the ratio input
      monthlyValues: Array(12).fill(1), // Array to hold monthly values (for each month of the year)
      start_year: 2011,
      stop_year: new Date().getFullYear() - 1,
      // Default values for the table

      currentCommune: null,
      currentDepartement: null,
      currentStartYear: null,
      currentStopYear: null,
      weather_data: new Map(),
      computed_data: { 'volume': Array(0), 'consumed': Array(0), 'needed': Array(0) },
      table_sum: new Map(),
      chart_data: { labels: [], datasets: [{ label: 'Volume d\'eau stocké dans la cuve (m3)', borderColor: "#000FAF", borderWidth: 1, backgroundColor: "#000FAF", pointRadius: 0, pointHitRadius: 10, data: [] }] },
      chart_option: {
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
        }
      }


    };
  },
  methods: {
    // Method to refresh the form  https://api.geosas.fr/edr/collections/safran-isba/position?coords=POINT(5.36 45.958)&datetime=2011-01-01/2024-12-31&parameter-name=PRELIQ_Q&crs=EPSG:4326&f=CoverageJSON
    actualiserForm() {
      if ((this.selectedCommune != this.currentCommune || this.selectedDepartement != this.currentDepartement || this.start_year != this.currentStartYear || this.stop_year != this.currentStopYear)) {
        if (this.selectedCommune == '' || this.selectedDepartement == '') {
          window.alert('Veuillez selectionner un département et une commune');
          return;
        }
        this.currentCommune = this.selectedCommune;
        this.currentDepartement = this.selectedDepartement;
        this.currentStartYear = this.start_year;
        this.currentStopYear = this.stop_year;
        var url = `https://api.geosas.fr/edr/collections/safran-isba/position?coords=POINT(${this.selectedLongitude} ${this.selectedLatitude})&datetime=${this.start_year}-01-01/${this.stop_year}-12-31&parameter-name=PRENEI_Q,PRELIQ_Q&crs=EPSG:4326&f=CoverageJSON`
        var xmlHttp = new XMLHttpRequest();
        xmlHttp.open("GET", url, false); // false for synchronous request
        xmlHttp.send(null);
        var response = JSON.parse(xmlHttp.responseText);
        if (response.response != undefined) {
          window.alert("Erreur lors de la requête.");
          return;
        }
        this.weather_data.clear();
        for (let i = 0; i < response.domain.axes.t.values.length; i++) {
          let date = response.domain.axes.t.values[i].slice(0, 10);
          let rain = response.ranges.PRELIQ_Q.values[i] + response.ranges.PRENEI_Q.values[i];
          this.weather_data.set(date, rain);
        }
        response = null;
      }
      if (this.weather_data.size == 0 || this.volume === null || this.surface === null || this.ratio === null || this.monthlyValues.includes('')) {
        window.alert('Erreur: Valeure manquante pour le calcul');
        return;
      }
      let i = 0;
      this.table_sum.clear();

      this.chart_data.labels = []
      this.chart_data.datasets[0].data = []

      this.weather_data.forEach((rain, date) => {
        let year = date.slice(0, 4);
        let month = parseInt(date.slice(5, 7), 10) - 1;
        if (i == 0) {
          this.computed_data.volume[i] = 0;
          this.computed_data.consumed[i] = 0;
          this.computed_data.needed[i] = 0;
        }
        else {
          let vol_inc = this.computed_data.volume[i - 1] + ((rain < 2 ? 0 : rain) * this.surface * this.ratio / 1000);
          this.computed_data.volume[i] = Math.max(0, Math.min(this.volume, vol_inc - this.monthlyValues[month]));
          this.computed_data.consumed[i] = Math.min(vol_inc, this.monthlyValues[month]);
          this.computed_data.needed[i] = Math.max(0, this.monthlyValues[month] - vol_inc);
        }

        this.chart_data.labels[i] = date;
        this.chart_data.datasets[0].data[i] = this.computed_data.volume[i];

        if (!this.table_sum.has(year)) {
          this.table_sum.set(year, Array(4).fill(0));
        }
        this.table_sum.get(year)[0] += this.computed_data.volume[i] == 0 ? 1 : 0;
        this.table_sum.get(year)[1] += this.computed_data.consumed[i];
        this.table_sum.get(year)[2] += this.computed_data.needed[i];

        i++;
      });
      let min = Array(4).fill(-1);
      let max = Array(4).fill(0);
      let mean = Array(4).fill(0);

      this.table_sum.forEach((a, _) => {
        a[3] = a[1] * 100 / (a[1] + a[2]);

        for (let j = 0; j < 4; j++) {
          min[j] = min[j] == -1 ? a[j] : Math.min(min[j], a[j]);
          max[j] = Math.max(max[j], a[j]);
          mean[j] += a[j];
        }
      });
      for (let j = 0; j < 4; j++) {
        mean[j] /= this.table_sum.size;
      }
      this.table_sum.set('Min', min);
      this.table_sum.set('Max', max);
      this.table_sum.set('Moyenne', mean);

      this.table_sum.forEach((v, k) => {
        for (let j = 0; j < 4; j++) {
          v[j] = parseFloat(v[j].toFixed(4));
        }
        v[3] += '%';
      });
      this.chart_data = {
        labels: [...this.chart_data.labels],
        datasets: [
          {
            ...this.chart_data.datasets[0],
            data: [...this.chart_data.datasets[0].data]
          }
        ]
      };
    }
  },
  computed: {
    uniqueDepartments() {
      return [...new Set(this.items.map(item => item.departement))]
        .sort((a, b) => {
          const numA = parseInt(a, 10);
          const numB = parseInt(b, 10);
          return numA - numB;
        });
    },
    filteredCommunes() {
      return this.items
        .filter(item => item.departement === this.selectedDepartement)
        .map(item => item.commune)
        .sort((a, b) => a.localeCompare(b));
    },
    selectedMairieCoordinates() {
      if (this.selectedCommune) {
        const selectedItem = this.items.find(item => item.commune === this.selectedCommune && item.departement === this.selectedDepartement);
        if (selectedItem) {
          return { latitude: selectedItem.latitude_mairie, longitude: selectedItem.longitude_mairie };
        }
      }
      return { latitude: null, longitude: null };
    }
  },
  watch: {
    selectedCommune(newCommune) {
      const { latitude, longitude } = this.selectedMairieCoordinates;
      this.selectedLatitude = latitude;
      this.selectedLongitude = longitude;
    }
  }
};
</script>

<template>
  <div class="container">
    <div class="input-section">
      <form>
        <!-- First Select for Departements -->
        <select v-model="selectedDepartement">
          <option value="" disabled selected>Select a Departement</option>
          <option v-for="departement in uniqueDepartments" :key="departement" :value="departement">{{ departement }}
          </option>
        </select>

        <!-- Second Select for Communes based on selected Departement -->
        <select v-model="selectedCommune" :disabled="!selectedDepartement">
          <option value="" disabled selected>Select a Commune</option>
          <option v-for="commune in filteredCommunes" :key="commune" :value="commune">{{ commune }}</option>
        </select>

        <div>
          <label> Année de debut / fin</label><br>
          <input class="year-input" type="number" v-model.number="start_year" placeholder="Enter Start Year" />
          <input class="year-input" type="number" v-model.number="stop_year" placeholder="Enter Stop Year" />
        </div>

        <!-- Number Inputs for Volume, Surface, and ratio -->
        <div>
          <label for="volume">Volume cuve (m<sup>3</sup>) : </label> <br>
          <input type="number" v-model.number="volume" id="volume" placeholder="Enter volume" min="0" /> <br>

          <label for="surface">Surface de toiture collectée (m<sup>2</sup>) : </label> <br>
          <input type="number" v-model.number="surface" id="surface" placeholder="Enter surface" min="0" /> <br>
          <div
            title="Taux de récupération de l'eau de toiture (dépend de la pente et de la nature du toit. Toit métallique : 0,9 ; toit en tuiles 0,8 ; Toit végétalisé : 0,4 ; terrasse gravillonnée : 0,55 ; Terrasse béton : 0,75). Le taux de récupération des pluies journalières inférieures à 2mm est fixé à zéro)">
            <label for="ratio">
              Taux de collecte <img src="./assets/info.png" width="16" height="16" /> : 
            </label> <br>
            <input type="number" v-model.number="ratio" id="ratio" placeholder="Enter ratio" min="0" max="1"
              step="0.1" />
          </div>
          <br>
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
        <button type="button" @click="actualiserForm" class="actualiser-btn">Actualiser</button>
      </form>
    </div>

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
            <tr v-for="(value, key) of table_sum" :key="rowIndex" :id="value[0]">
              <td :id="value[0]">{{ value[0] }}</td>
              <td :id="value[0]">{{ value[1][0] }}</td>
              <td :id="value[0]">{{ value[1][1] }}</td>
              <td :id="value[0]">{{ value[1][2] }}</td>
              <td :id="value[0]">{{ value[1][3] }}</td>
            </tr>
          </tbody>
        </table>
        <i>Calcul réalisé à partir des pluies journalières (Source : Météo France, modèle SAFRAN) données issues de
          l’API GEOSAS)</i>
      </div>
      <div class="chart-container">
        <Line :data="chart_data" :options="chart_option" />
      </div>
    </div>
  </div>
</template>

<style scoped>
.container {
  padding: 10px;
  height: 100%;
  width: 100%;
}

.input-section {
  /* Takes up half of the screen */
  width: 19%;
  padding: 1%;
  float: left;
  height: 100%;
  position: fixed;
  background-color: #FAFAFA;
  left: 0;
  top: 0;
}

.year-input {
  margin-bottom: 1px;
  padding: 1px;
  font-size: 1rem;
  width: 40%;
}

form {
  display: flex;
  flex-direction: column;
}

form select,
form input {
  margin-bottom: 15px;
  padding: 10px;
  font-size: 1rem;
}

.monthly-inputs h3 {
  margin-top: 20px;
  margin-bottom: 15px;
}

.months-container {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  /* 3 columns */
  gap: 15px;
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

.output-section {
  width: 80%;
  float: right;
  height: 100%;
  position: fixed;
  overflow: hidden;
  right: 0;
  top: 0;
}

.chart-container {
  width: 50%;
  float: left;
  padding: 10px;
}

.table-container {
  width: 45%;
  border: 1px solid #ddd;
  padding: 10px;
  float: left;
}

table {
  width: 100%;
  border-collapse: collapse;
}

table th,
table td {
  text-align: center;
  border: 1px solid #ddd;
}

table th {
  background-color: #f4f4f4;
}

#Min,
#Max,
#Moyenne {
  background-color: #E4E4E4;
  font-weight: bold;
}

h3,
label {
  font-weight: bold;
}
</style>
