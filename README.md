# Proyecto Júpiter — WhiteHosting

Análisis del mercado de alquiler vacacional en Madrid, Barcelona y Valencia, desarrollado como Trabajo Fin de Máster (Máster en Análisis de Datos, Pontia Tech, 2026).

## Contexto

WhiteHosting es un fondo de inversión (caso de estudio inspirado en el modelo de Leaf Living / Blackstone) que busca destinar **300 millones de euros** a la compra de alojamientos turísticos en Madrid, Barcelona y Valencia. Este proyecto responde, con evidencia de datos, a las preguntas planteadas en el caso: qué se puede comprar, a cuánto se puede alquilar, cuántos días al año y, en consecuencia, qué rentabilidad y qué estrategia de cartera resultan óptimas.

## Fuentes de datos

El análisis combina cuatro fuentes de datos complementarias:

| Fuente | Contenido | Volumen |
|---|---|---|
| **Dataset facilitado (Pontia)** | Características detalladas de inmuebles en alquiler vacacional (habitaciones, capacidad, licencia, etc.) | 5.551 inmuebles (5.252 tras limpieza) |
| **Scraping propio de Airbnb.es** (abr-jun 2026) | Precio de mercado actual; permite medir estacionalidad y volatilidad | 453.060 registros en bruto (193.530 tras limpieza) |
| **Inside Airbnb** (sept. 2025) | Ocupación estimada real (variable clave no disponible en otras fuentes) y precio en Madrid y Valencia | 51.021 inmuebles |
| **Idealista + Fotocasa** (scraping propio) | Universo de pisos en venta sobre el que se calcula el ROI365 y se construyen las carteras de inversión | 30.893 pisos (25.370 tras deduplicar) |

Con estas cuatro fuentes se cubre el ciclo completo de la inversión: precio de compra, precio de alquiler, ocupación real y, a partir de ahí, rentabilidad.

## Estructura del repositorio

```
proyecto-jupiter-whitehosting/
│
├── notebooks/
│   ├── 01_webscraping_idealista_fotocasa_airbnb.ipynb
│   ├── 02_mercado_compraventa_idealista_fotocasa.ipynb
│   └── 03_alquiler_rentabilidad_inversion.ipynb
│
├── data/                     # Datasets procesados, listos para Power BI
├── powerbi/                  # Cuadro de mando (.pbix)
├── docs/                     # Memoria del proyecto e informe ejecutivo
├── requirements.txt
├── LICENSE
└── .gitignore
```

### 1. `01_webscraping_idealista_fotocasa_airbnb.ipynb`
Desarrollo de los sistemas de extracción de datos de Idealista, Fotocasa y Airbnb. Documenta el proceso completo: primera aproximación con `requests` + `BeautifulSoup` (bloqueada con error HTTP 403), migración a Selenium, medidas anti-detección (configuración del navegador, modificación de propiedades JavaScript, temporización), y las distintas iteraciones hasta obtener extracciones estables por ciudad y barrio. Incluye también la combinación de los CSV de Idealista y Fotocasa por distrito.

### 2. `02_mercado_compraventa_idealista_fotocasa.ipynb`
Limpieza, depuración y armonización de los datos de compraventa (Idealista y Fotocasa): tratamiento de duplicados, valores nulos, outliers de precio/m², extracción de habitaciones/planta/ascensor, cálculo de precio/m² y precio/habitación, y armonización de barrios oficiales (incluyendo su cruce con Inside Airbnb). Finaliza con la segmentación de precios y la exportación de los datasets combinados.

### 3. `03_alquiler_rentabilidad_inversion.ipynb`
Notebook central del proyecto. Integra las cuatro fuentes, calcula la ocupación oficial y el beneficio neto anual (Ra) por barrio y tipología, construye el ROI365 sobre el universo de compraventa, diseña y compara los escenarios de inversión de 300M€, y responde una a una las preguntas del caso de estudio (sección "Case study WhiteHosting: respuestas").

## Metodología y hallazgos principales

### Cálculo del beneficio neto anual (Ra)

```
Ra = Oc × (Pn − Cn) × 365
```

- **Oc**: tasa de ocupación (0–69,9 %), tomada de `estimated_occupancy_l365d` de Inside Airbnb (estimación basada en reseñas reales; no se usa `availability_365` porque mide calendario abierto, no reservas efectivas).
- **Pn**: precio de análisis por noche (precio real por barrio y tipología, jerarquía de cuatro niveles).
- **Cn**: costes operativos, 35 % del precio (comisión de la plataforma, limpieza y carga fiscal).

Beneficio neto anual mediano por ciudad (modelo definitivo, precio por barrio):

| Ciudad | Beneficio neto mediano | Beneficio neto medio | n |
|---|---|---|---|
| Barcelona | 19.636 €/año | 20.847 €/año | 8.101 |
| Madrid | 9.282 €/año | 10.900 €/año | 11.844 |
| Valencia | 6.274 €/año | 7.903 €/año | 4.854 |

Estas cifras son deliberadamente conservadoras frente a las publicadas por fuentes de mercado (p. ej. AirROI 2026), ya que el modelo incluye todo el parque de alojamiento activo, sin filtrar por nivel de ocupación o calidad de gestión.

### Segmentación del mercado estándar vs. lujo

La frontera entre mercado estándar y segmento de lujo se calcula con el criterio de Tukey sobre el rango intercuartílico del precio por noche (scraping 2026, alojamientos completos): estándar hasta 408 €/noche, lujo entre 408 € y 592 €, outliers extremos por encima de 592 € (excluidos por no ser precios de mercado). La cartera de inversión se construye sobre el mercado estándar (95,1 % de la oferta) porque el segmento de lujo, pese a ser rentable por unidad, no tiene volumen suficiente para una cartera de 300M€ y concentra el riesgo en Barcelona, la ciudad con mayor exposición regulatoria.

### Rentabilidad de la compra (ROI365)

Sobre 25.370 pisos en venta deduplicados, el ROI365 mediano por ciudad es:

| Ciudad | ROI365 mediano |
|---|---|
| Barcelona | 3,10 % |
| Valencia | 2,14 % |
| Madrid | 1,56 % |

El ROI se calcula combinando el precio de alquiler mediano por barrio con el precio de compra real de cada inmueble, de modo que ambos términos son robustos frente a valores extremos.

### Estrategia recomendada para los 300 M€

Se comparan tres escenarios de cartera, con distinto horizonte de explotación en Barcelona (donde las licencias turísticas HUT se extinguen en 2028 según el PEUAT):

| Escenario | Inmuebles | Inversión | ROI conjunto | Beneficio neto anual | Exposición a Barcelona |
|---|---|---|---|---|---|
| E1 — Máximo ROI | 970 | 299,6 M€ | 4,70 % | 14,07 M€ | 45,1 % |
| **E2 — Sostenible (recomendado)** | **944** | **299,6 M€** | **4,08 %** | **12,23 M€** | **17,4 %** |
| E3 — Sin Barcelona | 866 | 299,8 M€ | 3,37 % | 10,09 M€ | 0 % |

Se recomienda el **escenario E2**: frente al máximo ROI teórico (E1), renuncia a 0,62 puntos de ROI y 1,84 M€ anuales a cambio de reducir en casi 28 puntos la exposición a la ciudad con mayor riesgo regulatorio. El reparto de capital por ciudad no se fija a mano, sino que se deriva del ROI mediano de los candidatos elegibles ponderado por los años de explotación restantes, con un límite de liquidez del 25 % del stock elegible en venta por ciudad.

Cartera E2 resultante: Barcelona (194 inmuebles, 52,0 M€, ROI 6,90 %), Madrid (319 inmuebles, 125,0 M€, ROI 3,74 %) y Valencia (431 inmuebles, 122,5 M€, ROI 3,23 %).

### Palanca de gestión operativa

Al margen de qué comprar, el hallazgo más accionable del análisis es operativo: el distintivo **Superhost** se asocia a entre 82 y 99 días más de ocupación anual, una mejora de rentabilidad que no depende del precio de adquisición del inmueble.

## Datasets exportados para Power BI (`data/`)

| Archivo | Contenido |
|---|---|
| `beneficio_mensual_ciudad_pbi.csv` | Beneficio bruto/neto mensual por ciudad (estacionalidad de la rentabilidad) |
| `beneficio_por_id_barrio_pbi.csv` | Precio, ocupación y beneficio neto mediano por barrio oficial |
| `cartera_inversion_300M_pbi.csv` | Cartera final de inmuebles seleccionados en el escenario recomendado (E2) |
| `cartera_por_escenario_pbi.csv` | Detalle de inmuebles para los tres escenarios comparados (E1, E2, E3) |
| `datos_compra_inmuebles_listo.csv` | Universo de compraventa limpio (Idealista + Fotocasa) |
| `estacionalidad_pbi.csv` | Precio medio/mediano por mes y ciudad (bruto y corregido) |
| `licencias_por_barrio_pbi.csv` | Porcentaje de anuncios con licencia turística por barrio |
| `licencias_por_ciudad_pbi.csv` | Porcentaje de anuncios con licencia turística por ciudad |
| `listing_precio_habitaciones_pbi.csv` | Listado de anuncios individuales con precio, ocupación e ingreso/beneficio anual estimado |
| `ocupacion_turistica_mensual.csv` | Ocupación turística oficial mensual por ciudad (fuente institucional) |
| `precio_por_barrio_turistico_pbi.csv` | Precio mediano en barrios turísticos objetivo |
| `precio_por_tipo_ciudad_pbi.csv` | Precio mediano por tipo de alojamiento y ciudad |
| `roi365_fotocasa_idealista_pbi.csv` | ROI365 calculado para el universo completo de compraventa |
| `tabla_barrios_con_coordenadas.csv` | Coordenadas (lat/long) de cada barrio oficial, para mapas en Power BI |
| `tabla_barrios_oficial_turisticos_id_mejorado.csv` | Tabla maestra de barrios oficiales, distrito, ciudad e indicador de barrio turístico |

> **Nota:** el archivo de trabajo "Tabla Barrios - armonizar turísticos.xlsx", usado internamente durante el proceso de armonización de barrios, no se incluye en este repositorio; su resultado final está recogido en `tabla_barrios_oficial_turisticos_id_mejorado.csv`.

## Cuadro de mando (`powerbi/`)

Dashboard de Power BI con la explotación visual de los datasets anteriores: mercado de alquiler por ciudad/barrio, estacionalidad, licencias turísticas, ROI365 y comparación de escenarios de inversión.

## Documentación (`docs/`)

- **Memoria del proyecto**: documento completo con planteamiento, metodología, análisis y conclusiones.
- **Informe ejecutivo**: resumen orientado a la toma de decisión de inversión.

## Limitaciones y consideraciones

- El scraping de Airbnb en Valencia corresponde a una única fecha con zonas concentradas en Ciutat Vella, por lo que no se usa como precio de referencia para el cálculo del Ra en esa ciudad (se usa en su lugar la mediana de Inside Airbnb).
- Inside Airbnb no incluye datos de precio para Barcelona, y su límite de publicación (285 $/noche) no recoge los alojamientos más exclusivos.
- El dataset Pontia (2017–2021) tiene un tope de 500 €/noche, por lo que no refleja el mercado actual ni sus segmentos más altos.
- La regulación turística solo se traduce en un filtro cuantitativo para Barcelona (horizonte de extinción datado, PEUAT); el requisito de acceso independiente en Madrid no puede verificarse con los datos de compraventa disponibles y se documenta como contexto, no como filtro.
- Las estimaciones de beneficio son deliberadamente conservadoras (incluyen todo el parque activo, no solo inmuebles optimizados), por lo que resultan inferiores a las de fuentes de mercado como AirROI.

## Stack técnico

Python (Pandas, NumPy), Selenium y BeautifulSoup para el scraping, Matplotlib/Seaborn para visualización exploratoria, y Power BI para el cuadro de mando final.

## Autoría

Proyecto desarrollado por Paula como parte del Trabajo Fin de Máster en Análisis de Datos (Pontia Tech, 2026), dentro del caso de estudio grupal "Proyecto Júpiter — WhiteHosting".
