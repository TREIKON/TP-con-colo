<template>
  <div class="container">
    <h1>Simulador Regulador 7812</h1>

    <!-- CONTROLES -->
    <div class="controls">
      <label>
        Vin: {{ vinValue.toFixed(2) }} V
        <input type="range" min="14" max="18" step="0.1" v-model.number="vinValue" />
      </label>

      <!-- CONFIGURAR PERTURBACIÓN DINÁMICA -->
      <div class="pert-box">
        <h3>Nueva Perturbación</h3>

        <label>
          Amplitud:
          <input type="number" max="5" step="0.1" v-model.number="newPertAmplitude" />
        </label>

        <label>
          Duración (s):
          <input type="number" step="0.1" v-model.number="newPertDuration" />
        </label>

        <button @click="addPerturbation">
          ➕ Agregar perturbación
        </button>
      </div>

      <div class="buttons">
        <button @click="startSimulation" :disabled="isRunning">▶ Iniciar</button>
        <button @click="pauseSimulation" :disabled="!isRunning">⏸ Pausar</button>
        <button @click="resetSimulation">🔄 Reiniciar</button>
      </div>
    </div>

    <!-- SLIDER HISTORIAL -->
    <div class="history-slider">
      <label>
        Desplazar historial
        <input
          type="range"
          :min="0"
          :max="maxWindowStart"
          v-model.number="windowStart"
        />
      </label>
    </div>

    <div class="buttons">
      <button @click="isMultiChart = !isMultiChart">📊 Cambiar Grafico</button>
    </div>

    <!-- MULTI CHART -->
    <div v-if="isMultiChart">
      <!-- VIN, VOUT -->
      <ChartComponent
        :height="230"
        :minYValue="0"
        :maxYValue="20"
        :labels="labels"
        :datasetLabels="[
          'Vin',
          'Vout',
        ]"
        :datasetData="[
          vinData,
          voutData,
        ]"
        :windowStart="windowStart"
        :windowSize="WINDOW_SIZE"
      />
      
      <!-- PERTURBACIÓN -->
      <ChartComponent
        :height="230"
        :minYValue="0"
        :maxYValue="5"
        :labels="labels"
        :datasetLabels="[
          'Perturbación'
        ]"
        :datasetData="[
          disturbanceData
        ]"
        :windowStart="windowStart"
        :windowSize="WINDOW_SIZE"
      />
      
      <!-- SENIALES -->
      <ChartComponent
        :height="230"
        :minYValue="0"
        :maxYValue="5"
        :labels="labels"
        :datasetLabels="[
          'Error',
          'Control'
        ]"
        :datasetData="[
          errorData,
          controlData
        ]"
        :windowStart="windowStart"
        :windowSize="WINDOW_SIZE"
      />
    </div>

    <!-- ALL IN ONE CHART -->
    <div v-else>
      <ChartComponent
        :height="690"
        :minYValue="0"
        :maxYValue="20"
        :labels="labels"
        :datasetLabels="[
          'Vin',
          'Vout',
          'Error',
          'Control',
          'Perturbación'
        ]"
        :datasetData="[
          vinData,
          voutData,
          errorData,
          controlData,
          disturbanceData
        ]"
        :windowStart="windowStart"
        :windowSize="WINDOW_SIZE"
      />
    </div>
  </div>
</template>

<script>
import { ref, computed } from "vue"
import ChartComponent from "./components/Chart.vue"

export default {
  components: { ChartComponent },

  setup() {

    // ================= PARÁMETROS =================
    const VREF = 12
    const KP = 1.2
    const DROP = 2
    const DT = 0.05
    const WINDOW_SIZE = 30

    // ================= ESTADOS =================
    const vinValue = ref(14)
    const vout = ref(0)
    const error = ref(0)
    const control = ref(0)

    const isRunning = ref(false)
    const isMultiChart = ref(false)
    let timer = null
    let time = 0

    const labels = ref([])
    const vinData = ref([])
    const voutData = ref([])
    const errorData = ref([])
    const controlData = ref([])
    const disturbanceData = ref([])

    const windowStart = ref(0)

    const maxWindowStart = computed(() =>
      Math.max(0, labels.value.length - WINDOW_SIZE)
    )

    const perturbations = ref([])
    const newPertAmplitude = ref(1)
    const newPertDuration = ref(0.1)

    function addPerturbation() {
      perturbations.value.push({
        start: time,
        duration: newPertDuration.value,
        amplitude: newPertAmplitude.value
      })
    }

    function getPerturbation(currentTime) {
      let total = 0
      perturbations.value.forEach(p => {
        if (currentTime >= p.start &&
            currentTime <= p.start + p.duration) {
          total += p.amplitude
        }
      })
      return total
    }

    function simulationStep() {
      time += DT

      const vin = vinValue.value
      const disturbance = getPerturbation(time)

      error.value = VREF - vout.value
      control.value = KP * error.value

      const dv = control.value + vin - DROP - disturbance - vout.value
      vout.value += 0.1 * dv

      if (vout.value > VREF + 0.2) vout.value = VREF + 0.2
      if (vout.value < 0) vout.value = 0

      labels.value.push(time.toFixed(2))
      vinData.value.push(vin)
      voutData.value.push(vout.value)
      errorData.value.push(error.value)
      controlData.value.push(control.value)
      disturbanceData.value.push(disturbance)

      if (windowStart.value >= maxWindowStart.value - 1) {
        windowStart.value = maxWindowStart.value
      }
    }

    function startSimulation() {
      if (isRunning.value) return
      isRunning.value = true
      timer = setInterval(simulationStep, 80)
    }

    function pauseSimulation() {
      isRunning.value = false
      clearInterval(timer)
    }

    function resetSimulation() {
      pauseSimulation()
      time = 0
      vout.value = 0

      labels.value = []
      vinData.value = []
      voutData.value = []
      errorData.value = []
      controlData.value = []
      disturbanceData.value = []
      perturbations.value = []

      windowStart.value = 0
    }

    return {
      vinValue,
      isRunning,
      isMultiChart,
      startSimulation,
      pauseSimulation,
      resetSimulation,
      windowStart,
      maxWindowStart,
      newPertAmplitude,
      newPertDuration,
      addPerturbation,
      labels,
      vinData,
      voutData,
      errorData,
      controlData,
      disturbanceData,
      WINDOW_SIZE
    }
  }
}
</script>