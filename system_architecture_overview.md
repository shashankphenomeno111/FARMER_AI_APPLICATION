# Project Sahyadri 2.0: System Architecture Overview

This document provides a high-level overview of the entire Project Sahyadri 2.0 architecture, detailing the data flow from ingestion to frontend visualization, the AI forecasting mechanics, and the spatial routing subsystem.

---

## 1. System Architecture Diagram

Below is the native Mermaid flowchart illustrating the end-to-end Project Sahyadri 2.0 system architecture:

```mermaid
graph TD
    subgraph "1. DATA INGESTION & PIPELINE"
        RawAPMC[Govt APMC Mandis JSON] & RawPrivate[Private Market Arrivals JSON] & RawIndustrial[Agro-Industrial Direct JSON] -->|Stream Ingest Script| SQLite[(SQLite: sahyadri_data.db)]
        SQLite -->|Exporter| Parquet[sahyadri_dataset.parquet]
    end

    subgraph "2. AI TRAINING & FORECASTING"
        Parquet -->|Load into Kaggle Notebook| Prep[Data Prep: Filter prices > 100, Cast strings]
        Prep -->|Weekly Sunday Resampling| Align[Continuous Timeline time_idx, ffill prices]
        Align -->|Pytorch Forecasting & Lightning| TFT[Temporal Fusion Transformer Model]
        TFT -->|Quantile Loss q=0.1, 0.5, 0.9| Checkpoint[sahyadri_tft_final.ckpt]
    end

    subgraph "3. SECURITY GATEWAY & ROUTING"
        Client[Client Browser] -->|HTTPS Port 443| Nginx[NGINX Reverse Proxy]
        Nginx -->|Strict Headers, TLS 1.2/1.3| Limiter[SlowAPI Rate Limiter]
        Limiter -->|JWT Verification| Auth[FastAPI Auth Handler]
        Auth -->|Route Request| FastAPI[FastAPI Controller]
    end

    subgraph "4. STANDALONE SERVER ENGINE (sahyadri_server.py)"
        FastAPI & Checkpoint -->|Lazy-Load CPU Singleton| Infer[TFT Model Inference Engine]
        Infer -->|Lookback Window & Future Horizon| Predict{Forecast Branching}
        Predict -->|History >= 15 weeks| TFT_Pred[TFT Quantiles Forecast]
        Predict -->|History < 15 weeks| Fallback[Heuristic Forecast: 1D Regression + Seasonal Multiplier]
        
        SQLite -->|Fetch nearest MSWC Godown & DCCB Branch| Geo[Spatial Database lookup]
        Geo -->|location_coords.json| Hav[Haversine Geodesic Distance Engine]
        Hav -->|Circuity Factor 1.25| Distance[Estimated Road Distance]
        
        FastAPI -->|Extract Name, Commodity, District| Chat[Multilingual Chat Processor]
        Chat -->|Regex Alias Mapper| Standardize[Devanagari Regex Entity Resolver]
        Standardize -->|Prompt Injection| Gemini[Gemini LLM API]
        Gemini -->|Translation Cache lookup| Cache[translated_cache.json]
    end

    subgraph "5. REACT FRONTEND DASHBOARD"
        Distance & TFT_Pred & Fallback & Cache -->|Compile JSON payload| React[React JS/JSX / Vite]
        React -->|Isometric Projection & Cubic Bezier curves| Ribbon[3D Ribbon Chart SVG]
        React -->|Farmer Slider Controls| Pledge[Live Pledge Finance Solver]
        React -->|Exact named business matching| Maps[Google Maps Directions URLs]
    end
```

---

## 2. Component Breakdowns & Workflows

### 2.1 Raw Data Channels & Pipeline Storage
* **Data Sources:** Raw JSON transactions are fetched dynamically from the **MahaAgriExchange (MahaAgX)** portal. This includes:
  * Government APMC Mandis transactions.
  * Private Market Arrivals.
  * Agro-Industrial Direct Corporate procurement records.
* **Storage Ingestion:** Raw JSON arrays are structured and inserted into the central SQLite database [sahyadri_data.db](file:///c:/Users/Shashank/OneDrive/Desktop/data_sets_fetched/market_and_transaction_data/sahyadri_data.db) under the `market_transactions` table.
* **Dataset Export:** To facilitate machine learning training on Kaggle/Colab, the data is exported into a compressed Parquet format `sahyadri_dataset.parquet` to ensure high-speed tensor loads.

### 2.2 AI Training & TFT Forecasting Inference
1. **Model Training:** The `sahyadri_dataset.parquet` dataset is ingested by a Kaggle/Colab notebook using GPU accelerators (Tesla T4 x2) to train the **Temporal Fusion Transformer (TFT)** model.
2. **Checkpoint Deployment:** The best weights are saved in a compiled checkpoint directory [sahyadri_trained_model[TFT_MODEL]/](file:///c:/Users/Shashank/OneDrive/Desktop/data_sets_fetched/market_and_transaction_data/sahyadri_trained_model[TFT_MODEL]) as `sahyadri_tft_final.ckpt`.
3. **Data Fetching for Forecasts:** When a query is initiated, the server fetches the historical transaction sequence for the selected market from the database:
   * **Weekly Resampling:** Daily transaction data is resampled to Sundays (using mean prices, minimum/maximum prices, and total quantities).
   * **Gap Handling:** Price gaps are forward-filled (`ffill`), and volume gaps are zero-filled (`0.0`).
   * **Lookback Padding:** The sequence is padded with a 12-week historical lookback (Encoder) and a 4-week future horizon (Decoder).
4. **Quantiles Prediction:** The model runs a forward CPU pass to predict three quantiles for the next 30 days:
   * **10th Percentile ($q=0.10$):** Minimum price boundary (worst-case).
   * **50th Percentile ($q=0.50$):** Modal price (most likely).
   * **90th Percentile ($q=0.90$):** Maximum price boundary (best-case).
5. **Linear + Seasonal Fallback:** If a market has insufficient history (<15 weeks), the server executes a fallback forecast using a 1D linear regression trend combined with crop-specific harvest seasonality coefficients (-7% during harvest peaks, +10% off-season).

### 2.3 Standalone Backend Server (sahyadri_server.py)
The Python server coordinates incoming API queries and executes spatial calculations:
* **Reads Coordinates:** Resolves coordinates for districts and mandis from [location_coords.json](file:///c:/Users/Shashank/OneDrive/Desktop/data_sets_fetched/market_and_transaction_data/location_coords.json).
* **Calculates Routes:** Uses the **Haversine Distance Engine** to calculate geodesic distance, multiplying by a circuity factor of **1.25** to estimate actual road travel distance in kilometers.
* **Extracts Entities:** Utilizes Devanagari and English regex patterns inside the Multilingual Chat Processor to identify names, crops, and districts from natural language questions.
* **Integrates Google Maps URLs:** Generates navigation URLs using exact business names (e.g., `APMC Market Nanded` or `MSWC Warehouse Udgir`) to ensure Google Maps routes to the exact physical business listing rather than a generic pin coordinate.

### 2.4 Frontend User Interface
* **Interactive Chat Mitra:** Renders a chat interface with dynamic crop recommendations and built-in AI safety disclaimers.
* **Interactive Dashboard:** Includes searchable filter dropdowns, crop seasonality calendars, and a pledge finance calculator.
* **3D Ribbon Chart:** Uses isometric projection coordinates and cubic Bezier curves to render price forecast trajectories dynamically in SVG.
