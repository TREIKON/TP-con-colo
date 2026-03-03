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
      <ChartComponent :height="180" :labels="labels" :datasetLabels="['Error']" :datasetData="[errorData]" :windowStart="windowStart" :windowSize="WINDOW_SIZE" />
    </div>

    <div v-else>
      <ChartComponent :height="540" :labels="labels" :datasetLabels="['Vin', 'Vout', 'Error', 'Perturbación']" :datasetData="[vinData, voutData, errorData, disturbanceData]" :windowStart="windowStart" :windowSize="WINDOW_SIZE" />
    </div>
  </div>
</template>

<script>
import { ref, computed } from "vue"
import ChartComponent from "./components/Chart.vue"

export default {
  components: { ChartComponent },
  setup() {
    const VREF = 12
    const DROP = 2
    const DT = 0.05
    const WINDOW_SIZE = 60

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

      // --- 1. ENTRADAS DEL SISTEMA ---
      // r: Valor de Referencia (Set-point). Lo que queremos alcanzar.
      const r = VREF; 
      
      // d1: Perturbación de Entrada (Tensión del transformador + Rizado).
      // No es la referencia, es una fuente de energía variable que el control debe rechazar.
      const ripple = showRipple.value ? 0.2 * Math.sin(2 * Math.PI * 4 * time) : 0;
      const d1_entrada = vinValue.value + ripple;

      // d2: Perturbación de Carga (Ruido eléctrico/Motores).
      const d2_carga = getPerturbation(time);

      // --- 2. LAZO DE CONTROL (Controlador Proporcional) ---
      
      // Señal de Error: e(t) = r(t) - y(t)
      // Representa la desviación de la salida respecto a la referencia de 12V.
      error.value = r - vout.value;

      // Acción de Control: u(t) = Kp * e(t)
      // Es la "señal necesaria" para corregir el sistema. 
      // Usamos un KP de 50 para un buen compromiso entre velocidad y estabilidad.
      let u = 50 * error.value;

      // --- 3. ACTUADOR Y SATURACIÓN (Límites Físicos) ---
      
      // El transistor de paso del 7812 no puede entregar más de lo que recibe 
      // menos su caída interna (V_drop).
      const limiteFisicoActuador = d1_entrada - DROP;
      
      // El control real aplicado (ua) se satura por la física del componente.
      control.value = Math.max(0, Math.min(u, limiteFisicoActuador));

      // --- 4. PLANTA / PROCESO (Modelo de la dinámica) ---
      
      // y(t): Salida del sistema (Vout).
      // Aplicamos un factor de suavizado (0.07) para simular la inercia de los 
      // capacitores de filtrado (evita los picos instantáneos molestos).
      const suavizado = 0.07; 
      
      // La fuerza neta que mueve la salida es el esfuerzo del control 
      // menos la perturbación que genera la carga.
      const fuerzaNeta = control.value - d2_carga;
      
      // Ecuación diferencial simplificada (Modelo de primer orden):
      const dy = fuerzaNeta - vout.value;
      vout.value += suavizado * dy;

      // --- 5. PROTECCIONES FINALES ---
      if (vout.value > limiteFisicoActuador) vout.value = limiteFisicoActuador;
      if (vout.value < 0) vout.value = 0;

      // --- 6. REGISTRO DE DATOS ---
      labels.value.push(time.toFixed(2))
      vinData.value.push(d1_entrada) // Visualizamos Vin como la perturbación de línea
      voutData.value.push(vout.value)
      errorData.value.push(error.value)
      controlData.value.push(control.value)
      disturbanceData.value.push(d2_carga)

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