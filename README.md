# Arquitectura multiagente con modelos de lenguaje para datos bananeros de Ecuador

*Multi-agent architecture with language models for Ecuadorian banana data: systematic review and evaluation*

[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

Manuscript ID: `<ID del manuscrito>`

## Autores y afiliaciones

- Joselyn Franco-Avila — Universidad Técnica de Machala, Ecuador
- Camilly Pacheco-Ordonez — Universidad Técnica de Machala, Ecuador
- Bertha Eugenia Mazon-Olivo — Universidad Técnica de Machala, Ecuador
- Maritza Pinta — Universidad Técnica de Machala, Ecuador

## 📁 Contenido del repositorio

### Estructura raíz

| Carpeta / Archivo | Descripción |
|---|---|
| `ETL_LlamaIndex_Banano_Experimento.ipynb` | Notebook de Databricks con el pipeline completo: tres agentes (extracción, ETL y exportación) coordinados con LlamaIndex Workflows y Meta Llama 4 Maverick, y la ejecución de la ingestión normal. |
| `requirements.txt` | Dependencias de Python (el notebook las instala con `%pip` en Databricks). |
| `LICENSE` | Licencia MIT. |
| `.gitignore` | Evita subir credenciales, datos y archivos exportados. |

> Los archivos fuente (ESPAC, SIPA, FAOSTAT y AEBE) **no se redistribuyen**: el agente de extracción los descarga desde los sitios oficiales.

### Estructura del notebook

| Bloque | Descripción |
|---|---|
| Bloque 0 | Reinicio opcional (borra tablas Delta y vacía volúmenes). Es destructivo. |
| Bloque 1 | Instalación de librerías. |
| Bloque 2 | Credenciales (Databricks Secrets), imports y configuración global. |
| Bloque 3 | Agente 1 · Extracción: listado, selección con el modelo, descarga y registro de métricas. |
| Bloque 4 | Tablas dimensionales (regiones y provincias de Ecuador). |
| Bloque 5 | Funciones de lectura, normalización y transformación por fuente. |
| Bloque 6 | Agente 2 · ETL: origen, lectura, mapeo, transformación con validación y carga a Delta Lake. |
| Ejecución | Ingestión normal: 10 corridas (R01–R10), verificación SQL y exportación de métricas (`xlsx` o `csv`). |
| Bloque 7 | Consultas de métricas de extracción, transformación y carga. |
| Bloque 8 | Agente 3 · Exportación de las tablas Delta a Google Drive. |

## 📁 Scripts

Este repositorio incluye el código necesario para reproducir la evaluación reportada en el manuscrito.

| Sección del notebook | Resultados relacionados | Descripción |
|---|---|---|
| Bloque 3 — Agente de extracción | Figura 4; Tabla 4 (selección de archivos) | Lista los archivos de cada fuente, selecciona con el modelo los que deben descargarse (con respaldo determinista), los descarga con control de hash y registra métricas por fuente. |
| Bloques 4 y 5 — Dimensiones y transformaciones | Figura 4 | Normaliza columnas, mapea provincias a identificadores y aplica la transformación especializada de cada fuente (ESPAC, SIPA, FAOSTAT y AEBE). |
| Bloque 6 — Agente ETL | Figura 5; Tabla 4 (mapeo al esquema e imputación de faltantes) | Resuelve el origen del archivo por reglas, lee, mapea al esquema (con el modelo), transforma con validación de calidad (el modelo decide si imputar faltantes con KNN, k = 5) y carga a Delta Lake; ante un fallo registra una fila ERROR y detiene solo ese archivo. |
| Ejecución — Ingestión normal (10 corridas) | Tabla 7, Figura 9 | Ejecuta 10 corridas del pipeline completo y registra tiempo, calidad, llamadas al modelo y estado de cada archivo en la tabla de control `control_logs_etl`. |
| Ejecución — Verificación SQL y exportación de métricas | Tabla 7 | Consulta las métricas por fuente y las exporta a Excel o CSV. |
| Bloque 7 — Métricas y consultas | Tabla 7 | Resúmenes por fuente de las métricas de extracción, transformación y carga. |
| Bloque 8 — Agente de exportación | Figura 4; Tabla 4 (prioridad de exportación); Figuras 6 a 8 (datos de entrada) | Decide con el modelo si exportar cada tabla y con qué prioridad, la convierte a Excel y la sube a Google Drive, de donde puede consumirse en herramientas de BI. Los tableros no se incluyen en este repositorio. |

## 🔁 Notas de reproducibilidad

- La evaluación corresponde a **ingestión normal**: 10 corridas sobre las cuatro fuentes originales, sin modificaciones, con 12 archivos por corrida (120 archivos en total).
- Modelo de lenguaje: `databricks-llama-4-maverick` (Meta Llama 4 Maverick) con `temperature=0`. Interviene solo en cuatro decisiones (Tabla 4): selección de archivos, mapeo al esquema, imputación de faltantes y prioridad de exportación; cada una tiene un respaldo determinista.
- El ejecutor limpia las tablas de datos y los volúmenes antes de comenzar y conserva los registros de la tabla de control.
- Todas las métricas se guardan en tablas Delta del esquema `bd_banano_ec`; las de la Tabla 7 salen de `control_logs_etl`.
- Las fuentes institucionales se actualizan de forma periódica, por lo que una nueva ejecución puede descargar archivos distintos y los valores pueden variar.
- Las credenciales (cuenta de servicio de Google Drive y FAOSTAT) se leen con `dbutils.secrets`; en el Bloque 2 del notebook se explica cómo crear el *secret scope* `banano-etl`.

## 📊 Resultados de referencia (Tabla 7 del manuscrito)

Desempeño por fuente en ingestión normal, 10 corridas de 12 archivos (120 archivos). Cada corrida duró entre 2,8 y 3,1 minutos (2,9 en promedio) y en ninguna ejecución se registró una fila con estado ERROR.

| Fuente y tabla de destino | Archivos | Tiempo medio, s (DE) | Registros válidos (%) | Llamadas al LLM por archivo |
|---|---|---|---|---|
| ESPAC (INEC): superficie y producción | 8 | 12,81 (1,80) | 100 | 1,00 |
| SIPA (MAG): uso del suelo | 1 | 7,91 (1,49) | 100 | 1,00 |
| SIPA (MAG): temperatura y precipitación | 1 | 25,74 (4,61) | 84,95 | 1,00 |
| FAOSTAT (FAO): producción de banano y plátano | 1 | 5,56 (0,91) | 100 | 1,00 |
| Bananotas (AEBE): exportaciones por región | 1 | 32,20 (0,87) | 100 | 0,00 |

DE: desviación estándar entre ejecuciones (n = 80 en ESPAC; n = 10 en las demás). En AEBE el destino se asigna por regla, sin consultar al modelo. Los valores se obtuvieron en Databricks Free Edition y pueden variar en otro entorno.

## 💻 Requisitos

- Workspace de Databricks con acceso a Foundation Models y a un Volume `databricksbanano.default.bronce` (subcarpetas `espac`, `sipa`, `faostat` y `aebe`)
- Python 3.12 (entorno de Databricks)
- LlamaIndex
- LangChain Databricks
- PySpark y Delta Lake (incluidos en Databricks)
- pandas, NumPy, scikit-learn
- pdfplumber, openpyxl, xlrd, BeautifulSoup4
- Google API Client (Drive)

Las dependencias se listan en `requirements.txt` (sin versiones fijadas). Reemplaza `<tu-usuario>` en las rutas `/Workspace/Users/...` del notebook.

## ⚠️ Aviso

Este repositorio es un prototipo de investigación. Un flujo que finaliza sin fallas no garantiza que los datos sean correctos: la validación previa a la carga es necesaria. El repositorio no contiene credenciales ni datos de las fuentes.

## ✉️ Contacto

Para consultas sobre replicación o uso académico:

Joselyn Franco-Avila (autora de correspondencia)
jfranco9@utmachala.edu.ec
Universidad Técnica de Machala
Ecuador
