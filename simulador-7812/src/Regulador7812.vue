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
          <input type="number" step="0.1" v-model.number="newPertAmplitude" />
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
          @input="updateChart"
        />
      </label>
    </div>

    <canvas ref="chartCanvas"></canvas>
  </div>
</template>

<script>
import { ref, onMounted, computed, watch } from "vue"
import Chart from "chart.js/auto"

export default {
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
    let timer = null
    let time = 0

    // ✅ HISTORIAL REACTIVO
    const labels = ref([])
    const vinData = ref([])
    const voutData = ref([])
    const errorData = ref([])
    const controlData = ref([])
    const disturbanceData = ref([])

    // ================= VENTANA =================
    const windowStart = ref(0)

    const maxWindowStart = computed(() =>
      Math.max(0, labels.value.length - WINDOW_SIZE)
    )

    // ================= PERTURBACIONES =================
    const perturbations = ref([])
    const newPertAmplitude = ref(1)
    const newPertDuration = ref(2)

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

    // ================= SIMULACIÓN =================
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

      // 🔥 Ahora sí es reactivo
      labels.value.push(time.toFixed(2))
      vinData.value.push(vin)
      voutData.value.push(vout.value)
      errorData.value.push(error.value)
      controlData.value.push(control.value)
      disturbanceData.value.push(disturbance)

      // Auto-scroll solo si estamos al final
      if (windowStart.value >= maxWindowStart.value - 1) {
        windowStart.value = maxWindowStart.value
      }

      updateChart()
    }

    function updateChart() {
      if (!chart) return

      const start = windowStart.value
      const end = start + WINDOW_SIZE

      chart.data.labels = labels.value.slice(start, end)
      chart.data.datasets[0].data = vinData.value.slice(start, end)
      chart.data.datasets[1].data = voutData.value.slice(start, end)
      chart.data.datasets[2].data = errorData.value.slice(start, end)
      chart.data.datasets[3].data = controlData.value.slice(start, end)
      chart.data.datasets[4].data = disturbanceData.value.slice(start, end)

      chart.update()
    }

    // ================= CONTROL =================
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
      perturbations.value = []

      windowStart.value = 0
      updateChart()
    }

    // ================= GRAFICO =================
    const chartCanvas = ref(null)
    let chart

    onMounted(() => {
      chart = new Chart(chartCanvas.value, {
        type: "line",
        data: {
          labels: [],
          datasets: [
            { label: "Vin", data: [] },
            { label: "Vout", data: [] },
            { label: "Error", data: [] },
            { label: "Control", data: [] },
            { label: "Perturbación", data: [] }
          ]
        },
        options: {
          animation: false,
          responsive: true,
          scales: {
            y: { title: { display: true, text: "Volt / Señal" } },
            x: { title: { display: true, text: "Tiempo (s)" } }
          }
        }
      })
    })

    // 🔥 Esto ahora sí funciona
    watch(windowStart, () => {
      updateChart()
    })

    return {
      vinValue,
      isRunning,
      startSimulation,
      pauseSimulation,
      resetSimulation,
      chartCanvas,
      windowStart,
      maxWindowStart,
      newPertAmplitude,
      newPertDuration,
      addPerturbation
    }
  }
}
</script>

<style scoped>
.container {
  max-width: 1000px;
  margin: auto;
  font-family: sans-serif;
}

.controls {
  display: flex;
  flex-direction: column;
  gap: 12px;
  margin-bottom: 20px;
}

.pert-box {
  border: 1px solid #ccc;
  padding: 10px;
  border-radius: 6px;
}

.history-slider {
  margin-bottom: 15px;
}

.buttons button {
  margin-right: 8px;
  padding: 6px 12px;
}
</style>