# 🔧 Simulador de Regulador 7812 – Vue 3

Este proyecto es un **simulador interactivo** del funcionamiento de un **regulador lineal 7812**, desarrollado con **Vue 3 + Vite + Chart.js**.

El objetivo es **didáctico**, orientado a nivel **facultad** (Electrónica / Teoría de Control), permitiendo visualizar:

- Regulación de tensión alrededor de 12 V
- Señal de error (comparador interno)
- Señal de control (transistor de paso)
- Respuesta dinámica (transitorios)
- Efecto de perturbaciones térmicas / carga
- Saturación y límites físicos
- Pausar, reanudar y reiniciar la simulación

---

## 🧠 Modelo conceptual

El regulador se modela como un **sistema realimentado de primer orden**:

- **Referencia (Vref)**: zener interno (~12 V)
- **Amplificador de error**: par diferencial interno
- **Control**: señal proporcional al error
- **Planta**: transistor serie + dinámica interna
- **Perturbación**: aumento de temperatura → mayor corriente → caída de Vout

No es una simulación SPICE, sino un **modelo dinámico simplificado**, ideal para análisis y defensa conceptual.

---

## 🖥️ Tecnologías utilizadas

- [Vue 3](https://vuejs.org/)
- [Vite](https://vitejs.dev/)
- [Chart.js](https://www.chartjs.org/)
- JavaScript

---

## 📁 Estructura del proyecto

```

simulador-7812/
├─ index.html
├─ package.json
├─ vite.config.js
└─ src/
├─ main.js
└─ App.vue
└─ Regulador7812.vue

````

### Instalar dependencias

Asegurate de tener **Node.js ≥ 18**.

```bash
npm install
```

Esto instalará:

* Vue 3
* Chart.js
* dependencias de Vite

---

### 3️⃣ Ejecutar el servidor de desarrollo

```bash
npm run dev
```

Abrí la [URL](http://localhost:5173/) en el navegador.

---

## 🎛️ Uso del simulador

### Controles disponibles

* **Slider Vin**
  Ajusta la tensión de entrada del regulador.

* **Slider Perturbación térmica**
  Simula aumento de temperatura.

* **▶ Iniciar**
  Comienza la simulación dinámica.

* **⏸ Pausar**
  Detiene la evolución temporal.

* **🔄 Reiniciar**
  Borra datos y vuelve al estado inicial.

---

### Señales mostradas en el gráfico

* **Vin**: tensión de entrada
* **Vout**: tensión regulada de salida
* **Error**: diferencia entre referencia y salida
* **Control**: señal que gobierna el transistor serie

El gráfico muestra una **ventana temporal limitada**, similar a un osciloscopio.

---

## 📊 Características técnicas

* Respuesta **no instantánea** (dinámica realista)
* Saturación superior e inferior
* Dropout modelado
* Limitación de puntos en el gráfico
* Actualización progresiva

---

## 🎓 Enfoque académico

Este simulador es ideal para:

* Explicar el funcionamiento interno del 7812
* Visualizar lazo de realimentación
* Analizar estabilidad y transitorios
* Defender trabajos prácticos de electrónica o control

---

## 🔮 Posibles extensiones

* Modo con / sin regulación
* Comparación regulador ideal vs real
* Zonas activa y saturada
* Diagramas internos interactivos
* Análisis de escalón formal

---

## 📜 Licencia

Uso libre con fines educativos.

---

## ✍️ Autor

Facundo Nahuel Dalsasso, Juan Cruz Rodriguez

Desarrollado como proyecto académico para simulación y análisis del regulador 7812.