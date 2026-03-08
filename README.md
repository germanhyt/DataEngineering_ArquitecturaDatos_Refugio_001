# 🌮 Refugio Data Cloud: Arquitectura y Automatización en GCP

> Arquitectura de solución en Google Cloud Platform (GCP) para modernizar y automatizar el ecosistema de datos del parque gastronómico "Refugio".

![GCP](https://img.shields.io/badge/Google_Cloud-4285F4?style=for-the-badge&logo=google-cloud&logoColor=white)
![Python](https://img.shields.io/badge/Python-3776AB?style=for-the-badge&logo=python&logoColor=white)
![BigQuery](https://img.shields.io/badge/BigQuery-669DF6?style=for-the-badge&logo=google-bigquery&logoColor=white)
![Power BI](https://img.shields.io/badge/Power_BI-F2C811?style=for-the-badge&logo=powerbi&logoColor=black)

## 📌 Contexto y Problema (AS-IS)
El parque gastronómico "Refugio" gestiona las ventas diarias de 21 locatarios. El proceso original dependía de un flujo ETL manual ejecutado localmente con scripts de Python y configuraciones basadas en Excel (`Configuracion.xlsx`). 

**Cuellos de botella identificados:**
* **Latencia y trabajo manual:** Dependencia de ejecuciones humanas semanales.
* **Límites de procesamiento:** Transformaciones complejas y consolidaciones realizadas en memoria RAM (`pandas`), limitando la escalabilidad.
* **Lógica acoplada:** Reglas de negocio (metas, algoritmos de proyección) incrustadas en código duro (*hardcoded*), bloqueando la autonomía de la gerencia.
* **Falta de resiliencia:** Ingesta directa desde Google Drive sin un Data Lake que respalde el histórico inmutable.

## 🎯 Objetivos de la Solución (TO-BE)
Migrar hacia una arquitectura **ELT (Extract, Load, Transform)** de grado empresarial, 100% nativa en la nube y basada en la **Arquitectura Medallón**.

1. **Automatización Zero-Touch:** Pipeline continuo sin intervención humana.
2. **Centralización Semántica:** Toda la transformación, limpieza y KPIs se calculan nativamente en **BigQuery SQL**.
3. **Data Lake Inmutable:** Resguardo de datos crudos en **Cloud Storage** a costo marginal.
4. **Analítica Avanzada:** Reemplazo de proyecciones estadísticas básicas por Machine Learning nativo (**BigQuery ML**).

---

## 🏗️ Arquitectura de la Solución

El siguiente diagrama ilustra el flujo de datos desde los puntos de venta hasta los dashboards analíticos:

```mermaid
graph LR
  classDef gcp fill:#e8f0fe,stroke:#4285f4,stroke-width:2px;
  classDef bq fill:#fce8e6,stroke:#ea4335,stroke-width:2px;
  classDef bi fill:#eefaf2,stroke:#34a853,stroke-width:2px;

  GD[Drive: CSVs Locatarios] -->|Lectura API| CF
  CS[Cloud Scheduler]:::gcp -->|Ejecución Diaria| CF[Cloud Function]:::gcp
  CF -->|Backup| GCS[(Cloud Storage)]:::gcp
  
  subgraph BigQuery [BigQuery - Arquitectura Medallón]
    Bronze[(Capa Bronze)]:::bq -->|Limpieza SQL| Silver[(Capa Silver)]:::bq
    GS[Sheets: Metas/Catálogos] -.->|Federated Query| Silver
    Silver -->|Agrupación/KPIs| Gold[(Capa Gold)]:::bq
    Gold -->|Entrenamiento| BQML((BigQuery ML)):::bq
    BQML -->|Proyección| Gold
  end
  
  CF -->|Inserta Data Cruda| Bronze
  Gold -->|Conexión Agnóstica| PBI[Power BI / Looker]:::bi