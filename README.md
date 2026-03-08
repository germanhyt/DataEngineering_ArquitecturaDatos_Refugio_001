# 🌮 Proyecto Final: Arquitectura de solución en GCP para modernizar y automatizar el ecosistema de Datos del parque gastronómico Refugio

![Google Cloud](https://img.shields.io/badge/GoogleCloud-%234285F4.svg?style=for-the-badge&logo=google-cloud&logoColor=white)
![Python](https://img.shields.io/badge/python-3670A0?style=for-the-badge&logo=python&logoColor=ffdd54)
![BigQuery](https://img.shields.io/badge/BigQuery-669DF6?style=for-the-badge&logo=google-bigquery&logoColor=white)
![Looker](https://img.shields.io/badge/Looker-4285F4?style=for-the-badge&logo=looker&logoColor=white)

## 1. Contexto y Problemática Actual (AS-IS)

La unidad de negocio retail gastronómico **“Refugio”** actualmente gestiona la información de ventas de más de 20 locatarios mediante un flujo de trabajo extractivo y transformacional (ETL) basado en la ejecución manual de scripts locales en Python y como almacenamiento de datos BigQuery.

Se han identificado los siguientes cuellos de botella y riesgos operativos:
* **Procesamiento Manual y Local:** La dependencia de ejecuciones semanales manuales genera retrasos en la disponibilidad de la información para la toma de decisiones.
* **Dependencia de Archivos Estáticos:** El uso de un archivo Excel local *core* de configuración para mapear columnas, gestionar estados de carga y definir dimensiones (Negocios, Turnos, Categorías) es frágil, propenso a corrupción y bloquea la concurrencia.
* **Lógica de Negocio Descentralizada:** Reglas críticas, como el cálculo de proyecciones estadísticas y la asignación de metas financieras, viven aisladas en scripts de Python fuera de la base de datos principal, dificultando su mantenimiento por parte de usuarios de negocio.
* **Cuello de Botella en el Procesamiento en Memoria:** Las transformaciones complejas (limpieza de formatos de fecha, normalización de moneda, agrupación de transacciones) se realizan en memoria usando Pandas, lo cual no es escalable a medida que aumente el volumen de tickets o el número de locatarios.

## 2. Objetivo del Proyecto

Diseñar e implementar una arquitectura de datos **ELT (Extract, Load, Transform)** en la nube (Google Cloud Platform), robusta, escalable y agnóstica a la herramienta de visualización final.

**Objetivos Clave:**
* **Automatización Total:** Eliminar procesos manuales en la ingesta de archivos (semanal/diaria).
* **Optimización FinOps:** Reducir costos de consultas en BigQuery mediante particionamiento y clusterización.
* **Gobernanza Centralizada:** Mover toda la lógica y KPIs a BigQuery (SQL) y usar Google Sheets para actualizar metas.
* **Resiliencia (Data Lake):** Crear respaldos inmutables de los datos crudos en Cloud Storage a un costo marginal.
* **Analítica Predictiva:** Usar **BigQuery ML** para pronósticos de ventas, reemplazando scripts locales.
* **Agnosticismo de BI:** Estructurar los datos para un consumo ultrarrápido en Power BI y facilitar una futura migración a Looker Studio.

## 3. Arquitectura de Solución (TO-BE)

La solución se fundamenta en la **Arquitectura Medallón (Medallion Architecture)**, utilizando servicios de Google Cloud para garantizar alta disponibilidad sin costos fijos de infraestructura.

![Diagrama de Arquitectura](https://res.cloudinary.com/dz0ajaf3i/image/upload/v1773000598/GCB/Captura_de_pantalla_2026-03-08_150850_xjmcyc.png)

### Fase 1: Ingesta y Data Lake (Extracción y Respaldo)
* **Interfaz de Usuario (Web -> Google Drive):** Los más de 20 locatarios continúan depositando sus cierres de caja en formato CSV/Excel en sus carpetas respectivas de repositorios mediante una página web.
* **Orquestación (Cloud Scheduler):** Actúa como el motor cronológico que dispara la ejecución del pipeline según la frecuencia requerida (plan 04:00 AM).
* **Microservicio de Extracción (Cloud Functions - Python):** Reemplaza el script local. Su única responsabilidad es conectarse a la API de Drive, identificar archivos nuevos y transferirlos al Data Lake.
* **Data Lake (Cloud Storage):** Almacenamiento de objetos inmutables y de ultra bajo costo. Los archivos se guardan estructurados por `/año/mes/día/locatario.csv`, garantizando un backup histórico.

### Fase 2: Data Warehouse - Arquitectura Medallón (Procesamiento ELT)
Todo el procesamiento se centraliza en BigQuery, eliminando cuellos de botella de memoria local.
* **Capa Bronze (Datos Crudos):** BigQuery lee los archivos desde Cloud Storage y los inserta como datos de tipo texto (`STRING`), agregando únicamente metadatos de linaje (fecha de carga, nombre de archivo). Actúa como la copia fiel del archivo original dentro de la base de datos.
* **Capa Silver (Limpieza y Normalización):** Mediante vistas y transformaciones SQL, se aplican las reglas de limpieza (corrección de fechas, limpieza de caracteres y estandarización de campos).
    * **Tablas Federadas:** Dimensiones como Catálogos de Negocios y Metas de Venta se leen en tiempo real directamente desde Google Sheets, dando autonomía operativa a la gerencia.
* **Capa Gold (Modelos de Negocio y Agregaciones):** Creación de métricas clave y operaciones SQL necesarias. 
    * **Optimización FinOps:** Tablas particionadas por fecha y clusterizadas por locatario para reducir drásticamente los costos de escaneo de datos.

### Fase 3: Analítica Avanzada y Consumo (Visualización)
* **Pronósticos con Inteligencia Artificial (BigQuery ML):** Dentro de la capa Gold, se plantea usar un modelo de series de tiempo (`ARIMA_PLUS`). BigQuery ML detectará automáticamente patrones de ventas por locatario y generará las proyecciones mensuales requeridas.
* **Capa de Presentación [Power BI ➔ Looker Studio]:** La herramienta de BI se conecta directamente a la capa Gold. Al estar todos los cálculos pre-procesados en BigQuery, la visualización en el dashboard será simple y de alto rendimiento.

---

## 🔄 Flujo de Datos

**1. Origen de Datos**
* Locatarios suben sus ventas (CSV/Excel) a **Google Drive**. Gerencia actualiza metas y catálogos en **Google Sheets**.

**2. Orquestación y Extracción**
* **Cloud Scheduler** activa el flujo de procesamiento automáticamente cada madrugada.
* Una **Cloud Function** detecta los archivos nuevos en Drive y guarda una copia inmutable en **Cloud Storage**.

**3. Capa Bronze (Ingesta en BigQuery)**
* La misma función carga los datos en bruto a **BigQuery**, añadiendo metadatos (fecha de carga y locatario).

**4. Capa Silver (Limpieza y Enriquecimiento)**
* Consultas automatizadas en BigQuery estandarizan formatos (fechas, monedas, métodos de pago) y cruzan los datos con Google Sheets.

**5. Capa Gold (Modelado y Predicción)**
* BigQuery agrupa las transacciones por ticket y calcula KPIs. Además, **BigQuery ML** procesa el histórico para proyectar las ventas a 30 días.

**6. Consumo (Dashboards)**
* **Power BI** (y a futuro Looker Studio) se conecta a la Capa Gold. Los dashboards cargan instantáneamente al consultar datos ya procesados.
