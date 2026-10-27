# PokaVision-IE v2 — DSS operacional híbrido para la inspección superficial de cajas de seguridad

Artefacto del PIA1 (Ingeniería Industrial, UPC). Integra un **Poka-Yoke de control** basado en visión
artificial con **trazabilidad digital por caja** y un **panel de soporte a la decisión** para una PyME
metalmecánica de Lima (empresa colaboradora, no identificada por acuerdo de confidencialidad).

| Componente DSS (Sprague, 1980) | Módulos | Decisión que apoya |
|---|---|---|
| Componente computacional | M1 Adquisición · M2 Procesamiento (ROI sin cerradura/herrajes) · M3 IA en dos etapas | — |
| Núcleo de decisión | M4 Poka-Yoke R1–R6 con dos umbrales τ1/τ2 calibrados por costo esperado | **Operacional**: liberar / retener / bloquear cada caja |
| Subsistema de datos | M5 Registro SQLite con hash SHA-256, dictamen con etapa de origen, compuerta de montaje, histórico CP0 | — |
| Herramienta analítica | M6 Pareto, gráfico p por etapa, matriz defecto–causa–acción · M7 Panel y acciones | **Táctica**: sobre qué etapa actuar |

## Instalación y uso

```bash
pip install -r requirements.txt
python generar_registros_proyectados.py  # registros del escenario proyectado
python datos_sinteticos.py          # CP1: 240 cajas, 960 imágenes, 64 combinaciones (SINTÉTICO)
python entrenar_modelo.py           # CP3: modelo + τ1/τ2 por costo + métricas
python corrida_prueba.py            # CP0 + CP5: corrida end-to-end con fechas dinámicas
python pruebas_funcionales.py       # CP0–CP6: 27 pruebas
python generar_datos_prueba.py      # imágenes demo por regla (data/demo)
python app.py                       # interfaz web Flask → http://localhost:5000
streamlit run app_streamlit.py      # interfaz alternativa (misma lógica)
```

Usuarios de demostración: `operario/1111`, `supervisor/2222`, `jefe/3333`, `montaje/4444`.

## Reglas del Poka-Yoke (M4)

| Regla | Condición | Estado |
|---|---|---|
| R1 | Imagen borrosa, oscura o de baja resolución | RETENIDO (repetir captura) |
| R2 | Metadatos incompletos (ID, lote, modelo, fase, vista) | RETENIDO |
| R3 | Puntaje de defecto s ≥ τ2 | BLOQUEADO (alerta + bloqueo de montaje) |
| R4 | τ1 ≤ s < τ2 | RETENIDO (dictamen del supervisor) |
| R5 | s < τ1 y verificaciones completas | LIBERADO |
| R6 | Soldadura visible con s ≥ τ1 | RETENIDO (verificar con método adecuado; la imagen no evalúa integridad) |

Solo una caja LIBERADA puede registrarse en montaje (compuerta, vista **Montaje**).

## Datos

* `data/dataset/`: imágenes **sintéticas** en formato YOLO (clases `rayadura`, `pintura_irregular`,
  `soldadura_visible`), separadas por caja (sin fuga entre train/val/test).
* `data/excel/Registros_Escenario_Proyectado_2023-2026.xlsx` (`python generar_registros_proyectados.py`, semilla 2026):
  3 980 registros por caja **generados** a partir del escenario proyectado del TF1 (Tabla 4.2), `Origen_dato = PROYECTADO`.
  Totales anuales fijos (1 053/112 · 1 151/129 · 1 250/150 · 526/64), tipos fijos en 2025 y en el total; no conformes
  muestreadas al azar (variación binomial entre meses), producción de lunes a viernes, lote mensual `L-AAAA-MM`,
  `Punto_deteccion = Inspección final` y `Etapa_origen_prueba` asignada por probabilidades de la Tabla 3.3 **solo para
  pruebas**. CP0 concilia el año 2025 (1 250 ID · 150 NC · S/ 90 000). No son mediciones de la empresa.
* `data/excel/Data_Maestra_4_Modelos.xlsx`: formato anterior (Data Maestra v3); el importador lo acepta, fija el punto
  de detección en la inspección final, deja vacía la etapa de origen y normaliza los ID a `CAJ-`.
* Para la validación con la empresa: reemplazar el dataset por fotos reales etiquetadas, entrenar YOLOv8
  (cuaderno Colab) y recalibrar τ1/τ2 con `entrenar_modelo.py`.

## Supuestos declarados

Ce = S/ 1 200 (escape, 2 × S/ 600), Cb = S/ 600 (falso bloqueo = costo de reproceso del escenario base), Cr = S/ 15 (revisión). Umbral de
nitidez de captura = 8 (varianza del Laplaciano): calibrar con fotos de la estación.
