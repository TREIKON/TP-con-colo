<template>
  <div class="container" style="padding: 20px; font-family: sans-serif; max-width: 1000px; margin: 0 auto;">
    <h1>Simulador Regulador 7812</h1>

    <div class="controls" style="background: #f4f4f4; padding: 20px; border-radius: 8px; margin-bottom: 20px;">
      <div style="margin-bottom: 20px; max-width: 300px;">
        <label style="display: block; font-weight: bold; margin-bottom: 5px;">
          Vin: {{ vinValue.toFixed(2) }} V
        </label>
        <input type="range" min="14" max="18" step="0.1" v-model.number="vinValue" style="width: 100%; cursor: pointer;" />
      </div>

      <div class="pert-box" style="border: 1px solid #ddd; padding: 15px; border-radius: 5px; margin-bottom: 20px; background: white;">
        <h3 style="margin-top: 0;">Nueva Perturbación</h3>
        <div style="display: flex; gap: 15px; align-items: center; flex-wrap: wrap;">
          <label>Amplitud: <input type="number" max="5" step="0.1" v-model.number="newPertAmplitude" style="width: 60px;" /></label>
          <label>Duración (s): <input type="number" max="1" step="0.1" v-model.number="newPertDuration" style="width: 60px;" /></label>
          <button @click="addPerturbation" style="padding: 5px 10px; cursor: pointer;">➕ Agregar</button>
        </div>
      </div>

      <div class="action-bar" style="display: flex; gap: 10px; flex-wrap: wrap; align-items: center; margin-bottom: 20px;">
        <button @click="startSimulation" :disabled="isRunning" style="background: #28a745; color: white; border: none; padding: 10px 15px; border-radius: 4px; cursor: pointer;">▶ Iniciar</button>
        <button @click="pauseSimulation" :disabled="!isRunning" style="background: #ffc107; border: none; padding: 10px 15px; border-radius: 4px; cursor: pointer;">⏸ Pausar</button>
        <button @click="resetSimulation" style="background: #dc3545; color: white; border: none; padding: 10px 15px; border-radius: 4px; cursor: pointer;">🔄 Reiniciar</button>
        
        <div style="height: 30px; width: 1px; background: #ccc; margin: 0 10px;"></div>

        <button @click="isMultiChart = !isMultiChart" style="background: #17a2b8; color: white; border: none; padding: 10px 15px; border-radius: 4px; cursor: pointer;">📊 Cambiar Gráfico</button>
        <button @click="showRipple = !showRipple" :style="{ background: showRipple ? '#e67e22' : '#6c757d', color: 'white', border: 'none', padding: '10px 15px', borderRadius: '4px', cursor: 'pointer' }">
          {{ showRipple ? '✅ Ripple ON' : '❌ Ripple OFF' }}
        </button>
      </div>

      <div class="history-slider" style="border-top: 1px solid #ccc; padding-top: 15px; padding-top: 15px;">
        <label style="display: block; font-weight: bold; margin-bottom: 5px;">
          Desplazar historial (Tiempo):
        </label>
        <input 
          type="range" 
          :min="0" 
          :max="maxWindowStart" 
          v-model.number="windowStart" 
          style="width: 100%; cursor: ew-resize;" 
        />
        <small style="color: #666;">Mové el slider para ver datos anteriores cuando la simulación avance.</small>
      </div>
    </div>

    <div v-if="isMultiChart">
      <ChartComponent :height="180" :labels="labels" :datasetLabels="['Vin', 'Vout']" :datasetData="[vinData, voutData]" :windowStart="windowStart" :windowSize="WINDOW_SIZE" />
      <ChartComponent :height="180" :labels="labels" :datasetLabels="['Perturbación']" :datasetData="[disturbanceData]" :windowStart="windowStart" :windowSize="WINDOW_SIZE" />
      <ChartComponent :height="180" :labels="labels" :datasetLabels="['Error', 'Control']" :datasetData="[errorData, controlData]" :windowStart="windowStart" :windowSize="WINDOW_SIZE" />
    </div>

    <div v-else>
      <ChartComponent :height="540" :labels="labels" :datasetLabels="['Vin', 'Vout', 'Error', 'Control', 'Perturbación']" :datasetData="[vinData, voutData, errorData, controlData, disturbanceData]" :windowStart="windowStart" :windowSize="WINDOW_SIZE" />
    </div>
  </div>
</template>

<script>
// ... (El script se mantiene igual al que pasaste, ya que contiene la lógica de maxWindowStart y windowStart)
import { ref, computed } from "vue"
import ChartComponent from "./components/Chart.vue"

export default {
  components: { ChartComponent },
  setup() {
    const VREF = 12
    const KP = 1.2
    const DROP = 2
    const DT = 0.05
    const WINDOW_SIZE = 30

    const vinValue = ref(14)
    const vout = ref(0)
    const error = ref(0)
    const control = ref(0)
    const isRunning = ref(false)
    const isMultiChart = ref(false)
    const showRipple = ref(false)
    
    let timer = null
    let time = 0

    const labels = ref([])
    const vinData = ref([])
    const voutData = ref([])
    const errorData = ref([])
    const controlData = ref([])
    const disturbanceData = ref([])

    const windowStart = ref(0)
    const maxWindowStart = computed(() => Math.max(0, labels.value.length - WINDOW_SIZE))

    const perturbations = ref([])
    const newPertAmplitude = ref(1)
    const newPertDuration = ref(0.1)

    function addPerturbation() {
      perturbations.value.push({ start: time, duration: newPertDuration.value, amplitude: newPertAmplitude.value })
    }

    function getPerturbation(currentTime) {
      let total = 0
      perturbations.value.forEach(p => {
        if (currentTime >= p.start && currentTime <= p.start + p.duration) total += p.amplitude
      })
      return total
    }

    function simulationStep() {
      time += DT

      const ripple = showRipple.value ? 0.2 * Math.sin(2 * Math.PI * 4 * time) : 0;
      const vin = vinValue.value + ripple;
      
      // 1. OBTENEMOS LA PERTURBACIÓN
      const rawDisturbance = getPerturbation(time);
      
      // 2. LIMITAMOS EL IMPACTO DE LA PERTURBACIÓN (Saturación de carga)
      // Esto evita que Vout se vaya a 0 a menos que sea un corto total.
      // Limitamos a que la carga no pueda restar más de lo que el sistema intenta subir.
      const maxAllowedImpact = (vin - DROP) * 0.8; 
      const effectiveDisturbance = Math.min(rawDisturbance, maxAllowedImpact);

      error.value = VREF - vout.value;
      control.value = KP * error.value;

      // 3. NUEVA ECUACIÓN DE DINÁMICA
      // Separamos la energía de entrada (Vin - DROP) de la corrección del control.
      const voutObjetivo = (vin - DROP) + (control.value * 0.5) - effectiveDisturbance;
      
      // dv es la velocidad con la que vout intenta llegar al objetivo
      const dv = voutObjetivo - vout.value;
      vout.value += 0.15 * dv; // Factor de suavizado (inercia de capacitores)

      // 4. SATURACIONES FINALES (Protecciones de Bolton)
      // Vout no puede ser mayor a la entrada menos el dropout
      if (vout.value > (vin - DROP)) vout.value = (vin - DROP);
      // Vout no puede superar el límite superior de seguridad
      if (vout.value > VREF + 0.5) vout.value = VREF + 0.5;
      if (vout.value < 0) vout.value = 0;

      // Guardado de datos
      labels.value.push(time.toFixed(2))
      vinData.value.push(vin)
      voutData.value.push(vout.value)
      errorData.value.push(error.value)
      controlData.value.push(control.value)
      disturbanceData.value.push(rawDisturbance)

      if (windowStart.value >= maxWindowStart.value - 1) {
        windowStart.value = maxWindowStart.value
      }
    }

    function startSimulation() {
      if (!isRunning.value) { isRunning.value = true; timer = setInterval(simulationStep, 80); }
    }
    function pauseSimulation() { isRunning.value = false; clearInterval(timer); }
    function resetSimulation() {
      pauseSimulation(); time = 0; vout.value = 0;
      labels.value = []; vinData.value = []; voutData.value = [];
      errorData.value = []; controlData.value = []; disturbanceData.value = [];
      perturbations.value = []; windowStart.value = 0;
    }

    return {
      vinValue, isRunning, isMultiChart, showRipple, startSimulation, pauseSimulation, resetSimulation,
      windowStart, maxWindowStart, newPertAmplitude, newPertDuration, addPerturbation,
      labels, vinData, voutData, errorData, controlData, disturbanceData, WINDOW_SIZE
    }
  }
}
</script>