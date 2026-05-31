# Project Sahyadri 2.0: Problem Statement & Datasets

This document details the agricultural challenges that **Project Sahyadri 2.0** solves for farmers in Maharashtra and outlines the core datasets cataloged from the **MahaAgriExchange (MahaAgX)** portal that power the platform.

---

## 1. Problem Statement

### The Agricultural Decision Dilemma
Maharashtra farmers face a persistent challenge when deciding how to handle their harvested produce. Specifically, they must answer two fundamental questions:
1. **When should I sell?** (Is it better to sell immediately or hold the crop?)
2. **Where should I sell?** (Which market—local APMC, private mandi, or corporate buyer—will yield the highest net return?)

### Key Pain Points
* **Information Asymmetry:** Farmers typically rely on isolated, fragmented channels for decision-making signals:
  * **Agmarknet** for spot prices.
  * **IMD (India Meteorological Department)** for weather forecasts.
  * **eNAM** for occasional transaction prices.
* **Lack of Fusion:** No existing public platform fuses field-level crop conditions, weather advisories, transport/logistics overheads, warehouse storage availability, and market price forecasts into a single, unified recommendation.
* **Distress Selling:** Due to immediate cash flow requirements and a lack of awareness of local credit options (like Warehouse Receipt Loans), farmers often engage in distress selling, losing significant potential revenue.
* **Financial Impact:** At a state scale, this lack of integrated market intelligence costs farmers a conservatively estimated **₹500 to ₹1,500 per quintal** on marginal selling decisions.

### Realistic Scenario
Consider a soybean farmer in Latur harvesting 4 hectares of produce:
* Without unified intelligence, the farmer sells immediately at the local Latur APMC.
* They are unaware that:
  1. The **Nanded APMC** modal price is projected to be **₹160/qtl higher** in two weeks.
  2. The **MSWC Warehouse** in Udgir has 1,400 tonnes of available storage space.
  3. A **DCCB (District Central Cooperative Bank)** branch nearby offers a **Warehouse Receipt Loan at 7% interest**, providing immediate liquidity to avoid distress selling.
* **The Sahyadri Solution:** Fuses these parameters, runs a mathematical optimization model, and recommends: *"Hold for 14 days, store at MSWC Udgir, take a DCCB warehouse receipt loan, and sell at Nanded APMC."* This results in an estimated upside of **₹6,400 per hectare**.

---

## 2. Core Datasets & Data Schemas

Project Sahyadri 2.0 utilizes **eight core datasets** curated and verified against the live **MahaAgriExchange (MahaAgX)** registry.

### 2.1 Market & Transaction Data

#### 1. Daily APMC Commodity Arrivals and Prices
* **MahaAgX Dataset ID:** `09739e2a-d809-11f0-b972-af914ab1614f`
* **Historical Range:** 2015 – Present
* **Key Fields / Schema:**
  * `apmc_name_eng` (Text) - Name of the APMC mandi.
  * `comm_name_eng` (Text) - Name of the commodity.
  * `variety_name_eng` (Text) - Variety of the commodity.
  * `min_rate`, `max_rate`, `modal_rate` (Numeric) - Price rates in INR per quintal (₹/qtl).
  * `arrivals` (Numeric) - Quantity of arrivals in quintals (qtl).
  * `r_date` (Date) - Arrival date.
* **Use Case:** Provides daily historical spot prices used to train the deep learning Temporal Fusion Transformer (TFT) forecasting model.

#### 2. Private Market Daily Commodity Arrivals and Prices
* **MahaAgX Dataset ID:** `474b0c02-cfb8-4a33-8d6e-2f419a0c847e`
* **Historical Range:** 2013 – Present
* **Key Fields / Schema:**
  * `user_name` (Text) - Registered private market/mandi name.
  * `crop_name` (Text) - Crop name.
  * `dist_name` (Text) - District name.
  * `min_rate`, `max_rate`, `model_rate` (Numeric) - Price rates in ₹/qtl.
  * `receipt_qty` (Numeric) - Quantity received in quintals.
* **Use Case:** Introduces private market arbitrage channels as alternatives to government-run APMCs.

#### 3. Agro-Industrial Direct Commodity Transactions
* **MahaAgX Dataset ID:** `7676296a-2d0c-4abb-b7db-a366795e8bb6`
* **Historical Range:** 2020 – Present
* **Key Fields / Schema:**
  * `cname` (Text) - Corporate buyer name.
  * `eng_name` (Text) - Commodity English name.
  * `dist_name` (Text) - District name.
  * `Value_in_Rs` (Numeric) - Transaction total value.
  * `Qty_in_Qtl` (Numeric) - Quantity transacted in quintals.
  * `Tran_date` (Date) - Transaction date.
* **Use Case:** Serves as a leading indicator of direct corporate buying demand and pricing options.

---

### 2.2 Infrastructure Registries (Distribution Rails)

#### 4. MSWC Commodity Warehouses
* **MahaAgX Dataset ID:** `8bad4a9a-0572-4ae1-98d8-ebc04e0f0e6b`
* **Format:** OGC GeoServer / Spatial DB
* **Key Fields / Schema:**
  * `plant_name` / `warehouse_name` (Text) - Name of the MSWC warehouse.
  * `capacity` (Numeric) - Total storage capacity in metric tonnes (MT).
  * `godown_type` (Text) - Owned vs. hired godowns.
  * `geometry` (WGS84 Point) - Latitude/Longitude coordinates.
  * `district_name`, `region` (Text) - Location context.
* **Use Case:** Enables local storage capacity checks and coordinates mapping for exact Google Maps routing.

#### 5. Agristack Bank Branches Registry
* **MahaAgX Dataset ID:** `6555aa5e-e473-439c-8cde-70886e4fc073`
* **Frequency:** Monthly Registry Updates
* **Key Fields / Schema:**
  * `bank_name`, `branch_name` (Text) - Financial institution details.
  * `ifsc_code` (Text) - Indian Financial System Code.
  * `district_name`, `address` (Text) - Geographic location of the branch.
  * `flag` (Boolean/Text) - Activity status (active/inactive).
* **Use Case:** Direct mapping of banking facilities to assist farmers with credit linkage.

#### 6. Agristack Cooperative Society Branches (DCCB)
* **MahaAgX Dataset ID:** `e13a1097-87c7-4ca0-bde5-d07a9f1a74aa`
* **Frequency:** Monthly Registry Updates
* **Key Fields / Schema:**
  * `dccb_name` / `society_name` (Text) - Name of the District Central Cooperative Bank branch.
  * `district_name`, `address` (Text) - Location context.
* **Use Case:** Maps localized DCCBs that provide critical financial services such as low-interest Warehouse Receipt Loans.

#### 7. Agristack Registry of Krishi Vigyan Kendras (KVK)
* **MahaAgX Dataset ID:** `2a9806ea-e132-4286-a734-ff3e75cafe9a`
* **Frequency:** Monthly Registry Updates
* **Key Fields / Schema:**
  * `kvk_name` (Text) - Name of the Krishi Vigyan Kendra.
  * `host_institute_of_kvk` (Text) - Managing agricultural institute or university.
  * `location` (GeoJSON Point) - Latitude/Longitude coordinates.
* **Use Case:** Identifies regional agronomic support centers for field trials and advisories.

#### 8. Agristack Soil Testing Laboratory Registry
* **MahaAgX Dataset ID:** `790ac3c2-9c5f-43e4-a58e-8456b7336eeb`
* **Historical Range:** 2023 – Present
* **Key Fields / Schema:**
  * `lab_type` (Text) - Government vs. private/cooperative laboratory.
  * `lab_capacity` (Numeric) - Number of soil samples testable per month.
  * `testing_parameters` (Text) - List of parameters measured (e.g., Nitrogen, Phosphorus, Potassium, pH, Electrical Conductivity).
  * `location` (Point) - Spatial coordinates.
* **Use Case:** Provides spatial data to support localized soil health analysis and crop advisory maps.
