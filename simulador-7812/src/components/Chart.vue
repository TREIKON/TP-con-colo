<template>
  <div :style="{ height: height + 'px', width: '100%' }">
    <canvas ref="chartCanvas"></canvas>
  </div>
</template>

<script>
import { ref, onMounted, watch } from "vue"
import Chart from "chart.js/auto"

export default {
  props: [
    "height",
    "minYValue",
    "maxYValue",
    "labels",
    "datasetLabels",
    "datasetData",
    "windowStart",
    "windowSize"
  ],

  setup(props) {

    const chartCanvas = ref(null)
    let chart

    function stringToColor(str) {
      let hash = 0

      for (let i = 0; i < str.length; i++) {
        hash = str.charCodeAt(i) + ((hash << 5) - hash)
      }

      const h = Math.abs(hash) % 360   // Hue (0-360)
      const s = 65                     // Saturación fija
      const l = 50                     // Luminosidad fija

      return `hsl(${h}, ${s}%, ${l}%)`
    }

    function updateChart() {
      if (!chart) return

      const start = props.windowStart
      const end = start + props.windowSize

      chart.data.labels = props.labels.slice(start, end)
      props.datasetData.forEach((dataArray, index) => {
        chart.data.datasets[index].data =
          dataArray.slice(start, end)
      })

      chart.update()
    }

    onMounted(() => {
      chart = new Chart(chartCanvas.value, {
        type: "line",
        data: {
          labels: [],
          datasets: props.datasetLabels.map((label) => {
            const color = stringToColor(label)

            return {
              label,
              data: [],
              borderColor: color,
              backgroundColor: color,
              tension: 0.2
            }
          })
        },
        options: {
          animation: false,
          responsive: true,
          maintainAspectRatio: false,
          scales: {
            y: { 
              min: props.minYValue,
              max: props.maxYValue,
              title: {
                display: true,
                text: "Volt / Señal"
              }
            },
            x: {
              title: {
                display: true,
                text: "Tiempo (s)"
              }
            }
          }
        }
      })
    })

    watch(
      () => [
        props.labels,
        props.datasetData,
        props.windowStart
      ],
      updateChart,
      { deep: true }
    )

    return { chartCanvas }
  }
}
</script>