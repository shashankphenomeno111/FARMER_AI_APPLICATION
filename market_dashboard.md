# Project Sahyadri 2.0: Interactive Market Dashboard Architecture

This document details the architecture, design, and mathematical model of the **Interactive Market Dashboard** in Project Sahyadri 2.0. The dashboard integrates live and forecasted market rates, vehicle logistics constraints, and pledge financing parameters to advise farmers on optimal sales strategies.

---

## 1. Dashboard Architectural Flow

The dashboard operates on a real-time reactive model. The diagram below illustrates how user inputs trigger server-side calculations, which are then visualised dynamically:

```mermaid
graph TD
    User([Farmer Input: Crop, District, Qty, Days]) -->|React State Update| A[Dashboard.jsx UI Controls]
    A -->|Debounced API Request| B[/api/recommend]
    B -->|FastAPI Controller| C[recommend_agent.py]
    C -->|1. Spatial DB Query| D[(Spatial DB: MSWC Warehouses, DCCB Banks)]
    C -->|2. TFT Model Inference| E[forecast_agent.py]
    C -->|3. Logistics Engine| F[Logistics Ledger & Vehicle Selector]
    C -->|4. Financial Solver| G[Net Revenue Optimizer]
    G -->|Structured JSON Response| H[Dashboard.jsx Render Cycle]
    H -->|Draw Isometric SVG Paths| I[3D Ribbon Price Chart]
    H -->|Map Location Search Terms| J[Google Maps Exact Directions Links]
    H -->|Calculate Sliders Real-time| K[Pledge Finance Solver]
```

---

## 2. Core Backend Recommendation Engine

The recommendation engine in [recommend_agent.py](file:///c:/Users/Shashank/OneDrive/Desktop/data_sets_fetched/market_and_transaction_data/backend/app/agents/recommend_agent.py) executes a multi-step financial optimization on each available market node for a selected commodity and district.

### 2.1 Spatial Infrastructure Matching
For a chosen district, the engine fetches nearby facilities:
1. **MSWC Warehouses:** Retrieves the closest warehouse, including its capacity, location, and real rental rate (`rent_per_qtl_month`).
2. **DCCB Bank Branches:** Retrieves the nearest cooperative bank branch, including its real interest rate (`interest_rate` p.a.).

### 2.2 Crop-Specific Finanical Mapping
If warehouse rents or bank interest rates are missing from the spatial database, the engine maps them dynamically based on the commodity:

| Commodity | Rental Rate (₹/qtl/month) | Loan-To-Value (LTV) Ratio |
| :--- | :--- | :--- |
| **Wheat, Rice, Maize, Bajri, Jawar, Cotton** | ₹10.00 | 80% (70% for Cotton) |
| **Gram, Pigeon Pea (Tur), Green Gram (Mug)** | ₹11.00 | 75% |
| **Soyabean (Soybean), Sugarcane, Groundnut** | ₹12.00 | 75% (60% for Sugarcane) |
| **Turmeric, Chilli** | ₹13.00 | 70% |
| **Potato** | ₹14.00 | 65% |
| **Onion** | ₹15.00 | 65% |
| **Tomato** | ₹18.00 | 50% |

### 2.3 Quantity-Based Vehicle Selection
To calculate transport costs, the system simulates real-world logistics parameters based on the transaction quantity:

| Quantity (Qtl) | Vehicle Option | Average Mileage | Visual Icon | Capacity Range |
| :--- | :--- | :--- | :---: | :--- |
| **0 – 10** | Mini Pickup | 12.0 km/L | 🚐 | 0 - 10 QTL |
| **10 – 30** | Pickup Truck | 8.0 km/L | 🚚 | 10 - 30 QTL |
| **30 – 80** | Medium Truck | 5.0 km/L | 🚚 | 30 - 80 QTL |
| **80 – 200** | Heavy Truck | 4.0 km/L | 🚛 | 80 - 200 QTL |
| **200+** | Multiple Heavy Trucks | 4.0 km/L (per truck) | 🚛 | `ceil(Qty / 200)` Trucks |
| **0 – 40** (Alt) | Tractor + Trolley | 3.5 km/L | 🚜 | Suitable if Distance $\le$ 50 km |

### 2.4 Mathematical Net Revenue Model
For each market node, the engine calculates the net payout under two scenarios:

#### Scenario A: Immediate Sell (Spot Today)
$$\text{Net Revenue}_{\text{Immediate}} = (\text{Spot Price} \times Q) - \text{Transport Cost} - \text{Mandi Fees}$$

* **Transport Cost:** $\text{Diesel Price (₹95.00/L)} \times \left( \frac{\text{Distance}}{\text{Mileage}} \right) \times N_{\text{trucks}} + \text{Toll} + 500.0$ (includes ₹500 base handling fee).
* **Toll and Mandi Fees by Market Type:**
  * **APMC Mandi:** Toll = ₹0.00, Mandi Fee = ₹45.00/qtl
  * **Private Market:** Toll = ₹50.00, Mandi Fee = ₹20.00/qtl
  * **Corporate Buyer:** Toll = ₹90.00, Mandi Fee = ₹0.00/qtl

#### Scenario B: Storing and Holding ($D \in \{7, 14, 30\}$ Days)
$$\text{Net Revenue}_{\text{Hold}(D)} = (\text{Forecasted Price}_D \times Q) - \text{Transport Cost} - \text{Mandi Fees} - \text{Rent}_D - \text{Interest}_D$$

* **Warehouse Rent ($\text{Rent}_D$):** $Q \times \text{Rent Rate (per qtl/month)} \times \left( \frac{D}{30} \right)$
* **DCCB Loan Value ($\text{Loan}$):** $\text{Spot Price} \times Q \times \text{LTV Rate}$
* **Cooperative Bank Interest ($\text{Interest}_D$):** $\text{Loan} \times \text{Interest Rate (p.a.)} \times \left( \frac{D}{365} \right)$

The engine compares all timeframes $D$ and selects the one that maximises net revenue. If holding yields an additional profit gain of at least **₹1,000** over immediate selling, the system outputs a **`HOLD`** strategy; otherwise, it advises **`SELL_NOW`**.

---

## 3. Interactive 3D Ribbon Chart Engine

To visualize the price trajectories of competing markets, [Dashboard.jsx](file:///c:/Users/Shashank/OneDrive/Desktop/data_sets_fetched/market_and_transaction_data/frontend/src/components/Dashboard.jsx) renders a premium **3D Ribbon Chart** using raw SVG paths and isometric projection.

### 3.1 Isometric Projection Formulas
The SVG canvas maps 3D coordinate space $(x, y, z)$ into 2D screen coordinates $(px, py)$ using trigonometric projection:
* **Horizontal Axis Tilt ($\theta_x = 12^\circ$):** Projects time index steps.
* **Depth Axis Tilt ($\theta_z = -22^\circ$):** Projects competing market ranks.
* **Vertical Axis ($y$):** Projects price scaling.

$$\begin{aligned}
px &= 75 + x \cos(\theta_x) + z \cos(\theta_z) \\
py &= 155 - y + x \sin(\theta_x) + z \sin(\theta_z)
\end{aligned}$$

### 3.2 Depth-Correct Z-Staggering
To prevent visual overlap when drawing ribbons for multiple markets, the frontend staggers each market's width along the depth axis ($z$) based on its net revenue rank:
* **Rank 0 (Best Market):** Projected between $Z_{\text{start}} = 16$ and $Z_{\text{end}} = 22$.
* **Rank 1 (Second Best):** Projected between $Z_{\text{start}} = 8$ and $Z_{\text{end}} = 14$.
* **Rank 2 (Third Best):** Projected between $Z_{\text{start}} = 0$ and $Z_{\text{end}} = 6$.

The rendering engine sorts the markets in descending order of depth (drawing Rank 2 first, and Rank 0 last) to ensure correct layer order.

### 3.3 Volumetric 3D Rendering with Cubic Bezier Curves
Instead of drawing jagged lines, the ribbon edges are calculated using **Cubic Bezier Curves** (SVG `C` commands). For each step from time step $i$ to $i+1$, control points are interpolated to smooth the curve:

```
           Control Point 1 (cp1)                 Control Point 2 (cp2)
           (cp1x = p0.x + dx/3, cp1y = p0.y)      (cp2x = p0.x + 2*dx/3, cp2y = p1.y)
                      o                                      o
                     /                                        \
  Start (p0) o------o                                          o------o End (p1)
```

The rendering engine combines these curves to construct volumetric meshes:
1. **Top Face (`topFaceD`):** Draws the upper surface of the ribbon, transitioning between $Z_{\text{start}}$ and $Z_{\text{end}}$.
2. **Front Face (`frontFaceD`):** Draws the vertical front edge of the ribbon down to the baseline ($y=0$).
3. **Volumetric Segments:** Draws lateral segments connecting time steps with gradients to give the ribbon visual depth.

---

## 4. UI Dashboard Panels

The frontend is divided into several interactive panels:

### 4.1 Searchable Filter Dropdowns
* **Component:** `SearchableSelect`
* **Features:** Contains custom filter inputs for crop and district. It features full-text query search and custom z-index stacking fixes to prevent dropdown panels from being clipped by cards behind them.

### 4.2 Crop Seasonality Calendar
* **Visual:** Displays a 12-month calendar card highlighting the crop's peak harvest months (e.g. October–November for Soybean, March–April for Wheat).
* **Purpose:** Warns farmers when local mandis are likely to be flooded with arrivals, which helps explain the model's downward price predictions during harvest season.

### 4.3 Pledge Finance Solver
* **Dynamic Recalculation:** Features interactive sliders for crop quantity (quintals) and holding duration (days). 
* **Calculations:** Instantly recalculates the ledger on the client side:
  * **Warehouse Storage Rent:** Calculated dynamically.
  * **DCCB Loan Value (Cash Advance):** Calculated dynamically using the crop's LTV ratio.
  * **Interest Accrued:** Calculated dynamically using the DCCB interest rate.
  * **Expected Net Gain:** Displays the net profit compared to immediate selling.

### 4.4 Logistics Ledger & Mapping
* **Ledger Details:** Renders a cost breakdown card for each market: Spot value, diesel cost, toll fees, mandi fees, handling fees, and expected net return.
* **Exact Routing Links:** Provides a button that links directly to Google Maps. The system constructs search queries using exact business names (e.g., `MSWC Warehouse Latur, ...` or `APMC Market Nanded, ...`) rather than generic coordinates, which prompts Google Maps to display the correct business listing and routes.
