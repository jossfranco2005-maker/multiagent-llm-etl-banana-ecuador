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
| `ETL_Experimento.ipynb` | Notebook de Databricks con el pipeline completo: tres agentes (extracción, ETL y exportación) coordinados con LlamaIndex Workflows y Meta Llama 4 Maverick, y la ejecución de la ingestión normal. |
| `requirements.txt` | Dependencias de Python (el notebook las instala con `%pip` en Databricks). |
| `LICENSE` | Licencia MIT. |
| `.gitignore` | Evita subir credenciales, datos y archivos exportados. |

> Los archivos fuente (ESPAC, SIPA, FAOSTAT y AEBE) **no se redistribuyen**: el agente de extracción los descarga desde los sitios oficiales.

### Estructura del notebook

| Componente / Etapa | Descripción |
|---|---|
| Configuración inicial | Reinicio opcional del entorno, instalación de librerías, configuración de credenciales (`Databricks Secrets`), importaciones y parámetros globales. |
| Agente 1 · Extracción | Listado de fuentes oficiales, selección de archivos mediante el modelo de lenguaje (con respaldo determinista), descarga con verificación de hash y registro de métricas. |
| Dimensiones y transformaciones | Normalización de columnas, generación de tablas dimensionales (regiones y provincias de Ecuador) y funciones especializadas de transformación por fuente (ESPAC, SIPA, FAOSTAT y AEBE). |
| Agente 2 · ETL y calidad | Resolución de origen por reglas, lectura, mapeo al esquema con LLM, validación de calidad, decisión de imputación de valores faltantes (KNN, k = 5) y carga a Delta Lake. |
| Ingestión normal y métricas | Ejecución de 10 corridas (R01–R10) del pipeline completo, registro en la tabla de control `control_logs_etl`, consultas SQL de métricas y exportación de resúmenes. |
| Agente 3 · Exportación | Evaluación de prioridad y exportación de tablas Delta a Excel y Google Drive para consumo en herramientas de BI. |

## 🔁 Notas de reproducibilidad

- La evaluación corresponde a **ingestión normal**: 10 corridas sobre las cuatro fuentes originales, sin modificaciones, con 12 archivos por corrida (120 archivos en total).
- Modelo de lenguaje: `databricks-llama-4-maverick` (Meta Llama 4 Maverick) con `temperature=0`. Interviene solo en cuatro decisiones: selección de archivos, mapeo al esquema, imputación de faltantes y prioridad de exportación; cada una tiene un respaldo determinista.
- El ejecutor limpia las tablas de datos y los volúmenes antes de comenzar y conserva los registros de la tabla de control.
- Todas las métricas se guardan en tablas Delta del esquema `bd_banano_ec`; las de evaluación salen de `control_logs_etl`.
- Las fuentes institucionales se actualizan de forma periódica, por lo que una nueva ejecución puede descargar archivos distintos y los valores pueden variar.
- Las credenciales (cuenta de servicio de Google Drive y FAOSTAT) se leen con `dbutils.secrets`; en la sección de configuración del notebook se explica cómo crear el *secret scope* `banano-etl`.

## 📊 Resultados de referencia

La preparación manual previa demandó entre 14 y 16 horas de trabajo (Tabla 7), y el prototipo resolvió sin intervención todas las tareas manuales que ahí se listan.

### Tabla 7. Esfuerzo de la preparación manual previa y del proceso automatizado

| Fuente | Archivos por corrida | Preparación manual previa (h)* | Tareas manuales que automatiza el prototipo |
|---|---|---|---|
| ESPAC (INEC) | 8 | 7 | Armar la plantilla, estandarizar resultados vacíos y unir banano y plátano en tabulados de varios años con orden distinto |
| SIPA (MAG) | 2 | 1 a 2 | Tratar la información y los duplicados |
| FAOSTAT (FAO) | 1 | 2 a 3 | Descargar desde el portal, eliminar columnas no útiles y limpiar |
| Bananotas (AEBE) | 1 | 4 | Extraer a tablas de Excel los boletines en PDF de varios años, hasta dos por año |
| **Total** | **12** | **14 a 16** | **Corrida automatizada completa: 4,2 min (de 4,0 a 4,4)** |

*Fuente: elaboración propia. \*Horas efectivas de trabajo estimadas, no mediciones cronometradas.*

Una corrida completa de las cuatro fuentes (12 archivos) tomó entre 4,0 y 4,4 minutos de reloj, con un promedio de 4,2, es decir, entre 200 y 230 veces menos que la preparación manual de esas fuentes. Esta comparación debe leerse con cautela: las horas manuales son estimaciones que incluyen la construcción inicial de las plantillas, así que una actualización manual posterior tomaría menos tiempo, y el tiempo automatizado no incluye el desarrollo de los agentes. Dentro de la corrida, la transformación, que incluye la consulta al modelo, ocupó cerca del 55 % del tiempo de proceso, frente a cerca del 30 % de la extracción y el 15 % de la carga.

En todos los archivos las filas validadas coincidieron con las insertadas en Delta Lake, y no se eliminaron duplicados. Las tablas conservaron todos sus registros, excepto la de temperatura y precipitación de SIPA, donde de 25.608 filas de entrada se cargaron 21.754 y el resto se descartó por las reglas de calidad antes de la carga; la causa de ese descarte no se analizó. Además, los 12 archivos produjeron exactamente el mismo número de filas en las diez corridas.

El modelo respondió con un JSON válido en las 120 llamadas y asignó la misma tabla de destino a cada archivo en las diez corridas (Tabla 8). En AEBE el destino se fijó por regla, sin consultar al modelo. La tipificación de columnas fue estable en SIPA y FAOSTAT, mientras que en ESPAC el modelo propuso columnas que no figuran en los archivos: nombres numéricos en el tabulado de 2019 y una columna denominada t2 en 2024 (diez de diez corridas) y, de forma intermitente, en 2020 y 2025 (una de diez). Estas propuestas no modificaron el número de filas ni su validez, aunque indican que, incluso con temperatura 0,0, la respuesta del modelo no siempre se repite, lo cual justifica que cada decisión cuente con un respaldo determinista. Además, la confianza reportada fue constante por fuente (0,95 en ESPAC y 0,90 en SIPA y FAOSTAT), por lo que no permite distinguir entre respuestas. De la selección de archivos solo se conservó el número de llamadas, no su razonamiento.

### Tabla 8. Comportamiento del modelo de lenguaje en las diez corridas de ingestión normal

| Decisión | Fuente | Resultado | Consistencia en 10 corridas | Observación |
|---|---|---|---|---|
| Tabla de destino | ESPAC, SIPA, FAOSTAT | Misma tabla para cada archivo | 100 % | SIPA uso del suelo se carga en la tabla de uso del suelo de ESPAC, cuya información SIPA también publica; AEBE se fija por regla |
| Tipos de columnas | SIPA | Decimales: precipitación, temperatura y superficie; enteros: año y mes | 100 % | Sin discrepancias detectadas |
| Tipos de columnas | FAOSTAT | Sin columnas que tipificar | 100 % | Ninguna |
| Tipos de columnas | ESPAC | Columnas ausentes del archivo: nombres numéricos (2019) y t2 (2020, 2024 y 2025) | 6 de 8 archivos idénticos | Sin efecto observado en filas ni en validez |
| Imputación (KNN) | SIPA clima | Sin imputación | 10 de 10 | En las demás fuentes la decisión no quedó registrada |
| Selección de archivos | Cuatro fuentes | Una llamada por fuente; ningún archivo omitido ni con error | 10 de 10 | No se conservó el razonamiento |

*Fuente: elaboración propia a partir de las respuestas que conserva cada ejecución en Delta Lake.*

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
