<template>
  <div class="container">
    <h1>Simulador Regulador 7812</h1>

    <!-- CONTROLES -->
    <div class="controls">
      <label>
        Vin: {{ vinValue.toFixed(2) }} V
        <input type="range" min="11" max="18" step="0.1" v-model.number="vinValue" />
      </label>

      <label>
        Perturbación térmica: {{ perturbacion.toFixed(2) }}
        <input type="range" min="0" max="2" step="0.05" v-model.number="perturbacion" />
      </label>

      <div class="buttons">
        <button @click="startSimulation" :disabled="isRunning">▶ Iniciar</button>
        <button @click="pauseSimulation" :disabled="!isRunning">⏸ Pausar</button>
        <button @click="resetSimulation">🔄 Reiniciar</button>
      </div>
    </div>

    <!-- GRAFICO -->
    <canvas ref="chartCanvas"></canvas>
  </div>
</template>

<script>
import { ref, onMounted } from 'vue';
import Chart from 'chart.js/auto';

export default {
  setup() {
    // ================== PARÁMETROS DEL MODELO ==================
    const VREF = 12;          // referencia zener
    const KP = 1.2;           // ganancia amplificador de error
    const DROP = 2;           // dropout
    const DT = 0.05;          // paso temporal
    const MAX_POINTS = 30;   // límite de puntos en gráfico

    // ================== ESTADOS ==================
    const vinValue = ref(14);
    const perturbacion = ref(0);

    const vout = ref(0);
    const error = ref(0);
    const control = ref(0);

    const isRunning = ref(false);
    let timer = null;
    let time = 0;

    // ================== DATOS ==================
    const labels = [];
    const vinData = [];
    const voutData = [];
    const errorData = [];
    const controlData = [];

    const chartCanvas = ref(null);
    let chart;

    // ================== MODELO DINÁMICO 7812 ==================
    function simulationStep() {
      time += DT;

      const vin = vinValue.value;
      const dist = perturbacion.value;

      // Amplificador de error (par diferencial interno)
      error.value = VREF - vout.value;

      // Etapa de ganancia y polarización
      control.value = KP * error.value;

      // Transistor serie
      const dv = (control.value + vin - DROP - dist - vout.value);
      vout.value += 0.1 * dv;

      // Saturaciones físicas
      if (vout.value > VREF + 0.2) vout.value = VREF + 0.2;
      if (vout.value < 0) vout.value = 0;

      // Guardar datos
      labels.push(time.toFixed(2));
      vinData.push(vin);
      voutData.push(vout.value);
      errorData.push(error.value);
      controlData.push(control.value);

      // Limitar puntos
      if (labels.length > MAX_POINTS) {
        labels.shift();
        vinData.shift();
        voutData.shift();
        errorData.shift();
        controlData.shift();
      }

      chart.update();
    }

    // ================== CONTROL DE SIMULACIÓN ==================
    function startSimulation() {
      if (isRunning.value) return;
      isRunning.value = true;
      timer = setInterval(simulationStep, 80);
    }

    function pauseSimulation() {
      isRunning.value = false;
      clearInterval(timer);
    }

    function resetSimulation() {
      pauseSimulation();
      time = 0;
      vout.value = 0;

      labels.length = 0;
      vinData.length = 0;
      voutData.length = 0;
      errorData.length = 0;
      controlData.length = 0;

      chart.update();
    }

    // ================== GRAFICO ==================
    onMounted(() => {
      chart = new Chart(chartCanvas.value, {
        type: 'line',
        data: {
          labels,
          datasets: [
            { label: 'Vin', data: vinData },
            { label: 'Vout', data: voutData },
            { label: 'Error', data: errorData },
            { label: 'Control', data: controlData }
          ]
        },
        options: {
          animation: false,
          responsive: true,
          scales: {
            y: { title: { display: true, text: 'Volt / Señal' } },
            x: { title: { display: true, text: 'Tiempo' } }
          }
        }
      })
    });

    return {
      vinValue,
      perturbacion,
      isRunning,
      startSimulation,
      pauseSimulation,
      resetSimulation,
      chartCanvas
    }
  }
}
</script>

<style scoped>
.container {
  max-width: 900px;
  margin: auto;
  font-family: sans-serif;
}
.controls {
  display: flex;
  flex-direction: column;
  gap: 12px;
  margin-bottom: 20px;
}
.buttons button {
  margin-right: 8px;
  padding: 6px 12px;
}
</style>
