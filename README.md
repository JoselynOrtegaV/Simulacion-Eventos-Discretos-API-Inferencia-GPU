# Simulación de Infraestructura API de Machine Learning con GPUs

Modelo de simulación de eventos discretos (DES) para evaluar la latencia y la gestión de recursos de una API de inferencia de visión por computadora en producción, usando SimPy.

---

## ¿Qué simula este proyecto?

Una API de clasificación de imágenes donde:

- Los **usuarios** envían peticiones HTTP con imágenes siguiendo un proceso de Poisson.
- Los **nodos GPU** ejecutan las inferencias con tiempos de servicio exponenciales.
- Cada inferencia consume **1 crédito de cómputo** (token de nube).
- El sistema usa una **política de reorden (s, Q)** para recargar créditos cuando el inventario cae al umbral `s`, con un Lead Time de respuesta del proveedor cloud.

---

## Parámetros operativos

| Parámetro | Valor | Descripción |
|---|---|---|
| λ (lambda) | 30 peticiones/min | Tasa de llegada de peticiones |
| μ (mu) | 10 imágenes/min por GPU | Tasa de servicio por nodo |
| c | 4 nodos GPU | Servidores paralelos |
| Stock inicial | 500 créditos | Tokens de cómputo disponibles |
| Punto de reorden s | 50 → 100 créditos | Umbral de recarga (optimizado) |
| Cantidad de recarga Q | 400 créditos | Tokens por recarga |
| Lead Time | ~2 min (σ=0.3) | Demora del proveedor cloud |
| Tiempo de simulación | 60 minutos | Duración de cada réplica |
| Réplicas | 30 | Para inferencia estadística |

---

## Estructura del notebook

```
1. Definición de parámetros          — Configuración del modelo M/M/c
2. Construcción del modelo DES        — Clase ApiML + procesos SimPy
3. Corrida visual (1 réplica)         — Gráficos de dientes de sierra y Wq
4. Evaluación operacional (30 réplicas) — IC 95%, Wq, predicciones fallidas
5. Visualización de réplicas          — Histogramas y barras por réplica
6. Análisis de predicciones fallidas  — ¿Por qué falla con ρ = 75%?
7. Corrección del punto de reorden    — Cálculo y justificación de s = 100
8. Comparación final s=50 vs s=100    — Impacto en calidad de servicio
```

---

## Resultados clave

### Con s = 50 (configuración original)

| Métrica | Valor |
|---|---|
| Wq media | ~3.24 segundos |
| IC 95% de Wq | [2.91 , 3.56] s |
| Predicciones fallidas (media) | ~65 por sesión de 60 min |
| Utilización teórica ρ | 75% |

### Con s = 100 (configuración optimizada)

| Métrica | Valor |
|---|---|
| Wq media | ~2.99 segundos |
| IC 95% de Wq | [2.49 , 3.48] s |
| Predicciones fallidas (media) | **0 por sesión de 60 min** |
| Utilización teórica ρ | 75% |

La latencia de cola se mantiene prácticamente idéntica entre ambas configuraciones, confirmando que el cuello de botella era el inventario, no el hardware.

---

## Por qué falla el sistema con s = 50

La utilización de los GPUs (ρ = 75%) mide exclusivamente la ocupación del hardware y es independiente del inventario de créditos. El problema es una **ventana de vulnerabilidad** durante el Lead Time:

1. Los créditos caen a 50 → se activa la recarga.
2. El proveedor tarda ~2 minutos en entregar los tokens.
3. Durante esos 2 minutos, el sistema consume ~60 créditos (30 peticiones/min × 2 min).
4. Como 50 < 60, los créditos se agotan antes de que llegue la recarga → predicciones fallidas.

---

## Cálculo del punto de reorden óptimo

El nuevo valor `s = 100` se compone de dos elementos:

- **Consumo durante Lead Time:** 30 peticiones/min × 2 min = **60 créditos**
- **Stock de seguridad** (z=3 para cobertura del 99.9%): 3 × 30 × 0.3 = **27 créditos**
- **Punto de reorden teórico:** 60 + 27 = 87 créditos → redondeado a **s = 100** para absorber escenarios extremos

---

## Instalación y ejecución

```bash
pip install simpy numpy pandas matplotlib scipy
jupyter notebook Simulacion_API_ML_GPU_v2.ipynb
```

Ejecutar las celdas en orden. Las 30 réplicas tardan aproximadamente 10-20 segundos en completarse.

---

## Dependencias

| Librería | Uso |
|---|---|
| `simpy` | Motor de simulación de eventos discretos |
| `numpy` | Cálculos estadísticos y generación de números aleatorios |
| `scipy` | Intervalo de confianza con distribución t de Student |
| `matplotlib` | Gráficos de dientes de sierra, histogramas y comparaciones |
| `pandas` | Manejo de datos (importado para extensibilidad) |
