# Haat_Bazar (হাট বাজার) — Engineering Specification & System Architecture

Haat_Bazar is a direct-from-farm marketplace engineered to disintermediate agricultural supply chains. The platform connects farmers directly with retail consumers (B2C) and commercial bulk buyers/wholesalers (B2B). By introducing automated minimum order thresholds, route-aggregated micro-fulfillment, and transparent fee structures, Haat_Bazar prevents farm-gate distress sales while delivering verified fresh produce to buyers.

---

## 1. Domain Context & Problem Engineering

### 1.1 Structural Inefficiencies in Traditional Agri-Supply Chains
In traditional South Asian and emerging agricultural markets (e.g., Bangladesh, India):
- **Intermediary Sprawl:** Produce moves through 4 to 6 layers of intermediaries (Local Faria $\rightarrow$ Bepari $\rightarrow$ Aratdar/Commission Agent $\rightarrow$ Wholesaler/Paikar $\rightarrow$ Retailer).
- **Price Dilution:** Farmers typically capture only **$20\% \text{ to } 35\%$** of the retail end-consumer price, while middle layers absorb up to **$50\% \text{ to } 65\%$** in margins and transactional overhead.
- **Post-Harvest Loss (PHL):** Transit delays, lack of cold-chain aggregation, and multiple handling stages result in **$25\% \text{ to } 40\%$** spoilage of perishable greens, tubers, and fruits.
- **Distress Selling:** Due to lack of real-time price discovery and immediate market access, smallholder farmers often dump inventory below marginal production cost during localized harvest gluts.

### 1.2 The Haat_Bazar Solution
- **Algorithmic Disintermediation:** Direct matching of farm harvest schedules with aggregated retail and wholesale demand.
- **Dynamic Threshold Protection (MOQ Enforcement):** Dynamic, algorithmic calculation of Minimum Order Quantities (MOQ) and Minimum Order Value (MOV) to ensure unit economics cover harvest, packaging, and local transport costs without farmer losses.
- **Platform Monetization Model:** Flat or percentage-based service fee + distance-and-weight-tiered fulfillment/delivery fee.

---

## 2. Platform Actors & Workflows

### 2.1 Actors & Personas
1. **Farmer (Producer/Seller):**
   - Profile: Low digital literacy, mobile-first or USSD/IVR assisted.
   - Operations: Posts upcoming or ready harvests, sets available quantity, sets baseline reservation price (or adopts recommended market price).
2. **Retail Consumer (B2C):**
   - Profile: Urban households seeking farm-fresh, chemical-traceable produce.
   - Operations: Purchases basket-based groceries subject to category minimums or aggregate basket minimums.
3. **Wholesaler / Institutional Buyer (B2B):**
   - Profile: Restaurants, hotels, supermarkets, and secondary-tier market traders.
   - Operations: Places bulk advance orders (e.g., metric tons or multi-sack lots) with scheduled terminal pickups or truckload logistics.
4. **Platform / Operations (Haat_Bazar Admin & Hub Logistics):**
   - Operations: Manages quality verification at micro-collection hubs, route planning, logistics dispatch, and escrow clearance.

---

## 3. Core Operational Logic & Mathematics

### 3.1 Pricing & Revenue Mechanics
For any executed order transaction, the total order amount billed to the buyer ($P_{\text{buyer}}$) and payout to the farmer ($P_{\text{farmer}}$) are governed by:

$$P_{\text{farmer}} = \sum_{i=1}^{n} (Q_i \times U_i)$$

Where:
- $Q_i$ = Quantity of item $i$ (in kg, crate, or bunch units).
- $U_i$ = Unit farm-gate price set or accepted by the farmer.

The total price charged to the buyer is:

$$P_{\text{buyer}} = P_{\text{farmer}} + F_{\text{service}} + F_{\text{delivery}} + T_{\text{tax}}$$

Where:
- **Service Fee ($F_{\text{service}}$):**
  $$F_{\text{service}} = \max\left(S_{\text{min}},\, P_{\text{farmer}} \times \alpha\right)$$
  - $\alpha$ = Platform commission percentage (e.g., $5\% \text{ to } 8\%$ for B2C; $2\% \text{ to } 4\%$ for B2B).
  - $S_{\text{min}}$ = Base operational platform fee.
- **Delivery Fee ($F_{\text{delivery}}$):**
  $$F_{\text{delivery}} = B_{\text{fee}} + (d \times C_{\text{dist}}) + (w \times C_{\text{weight}})$$
  - $B_{\text{fee}}$ = Base drop charge.
  - $d$ = Transit distance from collection hub to buyer destination.
  - $w$ = Billable weight of the aggregated basket.
  - $C_{\text{dist}}, C_{\text{weight}}$ = Variable unit cost coefficients.

---

### 3.2 Dynamic Minimum Threshold Engine (Anti-Loss Guard)
To preserve the market status quo and prevent farmers from incurring logistical losses on small, fragmented orders, the checkout validation enforces two gating constraints:

1. **Basket-Level Minimum Order Value (MOV):**
   $$\sum_{i=1}^{n} (Q_i \times U_i) \ge \text{MOV}_{\text{tier}}$$
   - $\text{MOV}_{\text{B2C}} \ge \text{BDT } 500 \text{ / } \$5.00$ (ensures packing and last-mile dispatch viability).
   - $\text{MOV}_{\text{B2B}} \ge \text{BDT } 5000 \text{ / } \$50.00$ (bulk wholesale minimum).

2. **Per-SKU Minimum Order Quantity (MOQ):**
   For bulky or low-margin crops (e.g., potatoes, onions, cabbage), individual items carry an item-level constraint:
   $$Q_i \ge \text{MOQ}_i$$
   - *Example:* Leafy greens ($MOQ = 1\text{ kg}$), Potatoes ($MOQ = 5\text{ kg}$ for retail, $50\text{ kg}$ for wholesale).

If $Q_i < \text{MOQ}_i$ or $\sum (Q_i \times U_i) < \text{MOV}$, the transactional state machine halts checkout and suggests automated basket-fill items to reach threshold viability.

---

## 4. System Architecture & Component Design

```
+-----------------------------------------------------------------------------------+
|                                Client Applications                                |
|  - Farmer Mobile App (Lightweight, Offline-First)                                  |
|  - Consumer & Wholesaler Web / Mobile Interface                                   |
|  - Hub Manager PWA (Barcode / Weigh-scale Integrated)                             |
+------------------------------------------+----------------------------------------+
                                           |
                                           v
+-----------------------------------------------------------------------------------+
|                            API Gateway & Routing Layer                            |
|       - Authentication (JWT / OTP)     - Rate Limiting     - Request Routing      |
+------------------------------------------+----------------------------------------+
                                           |
    +------------------+-------------------+-------------------+--------------------+
    |                  |                   |                   |                    |
    v                  v                   v                   v                    v
+---------+      +-----------+       +-----------+       +-----------+        +-------------+
| Product |      | Order &   |       | Dynamic   |       | Logistics |        | Settlement  |
| Catalog |      | Checkout  |       | Pricing & |       | & Route   |        | & Escrow    |
| Service |      | Service   |       | Threshold |       | Hub Engine|        | Service     |
+----+----+      +-----+-----+       +-----+-----+       +-----+-----+        +------+------+
     |                 |                   |                   |                     |
+----+-----------------+-------------------+-------------------+---------------------+--+
|                                    Data Layer                                     |
|  - Relational Core: PostgreSQL (Transactional consistency, Orders, Ledgers)       |
|  - Document Store: MongoDB (Catalog flexibility, Farmer attribute profiles)       |
|  - In-Memory Cache: Redis (Stock counters, Session locks, Geolocation caches)    |
|  - Message Broker: Apache Kafka / RabbitMQ (Event-driven asynchronous updates)     |
+-----------------------------------------------------------------------------------+
```

### 4.1 Service Breakdown

#### 1. Product Catalog & Harvest Service
- Handles SKU metadata, shelf-life indicators, grading standards (Grade A, B, C), and harvest timelines.
- Manages inventory using an **Available-to-Promise (ATP)** model:
  $$\text{Inventory}_{\text{ATP}} = \text{Harvested}_{\text{Verified}} + \text{Projected}_{\text{Yield}} - \text{Allocated}_{\text{Reservations}}$$

#### 2. Order & Threshold Validation Engine
- Evaluates B2C vs. B2B buyer profiles.
- Validates MOQ and MOV constraints in real time against inventory locks to prevent race conditions during flash gluts.
- Atomic reservation via Redis distributed locks (`Redlock`) to ensure parallel checkouts do not oversell finite farm batches.

#### 3. Logistics & Hub Aggregation Engine
- Groups orders by geographical clusters (Upazila/Block level hubs $\rightarrow$ City distribution centers).
- Computes aggregated farm pickup runs: Instead of dispatching single couriers to remote fields, regional logistics consolidates produce at local collection points (**Haat Hubs**) before trunk-line transport to urban hubs.

#### 4. Financial Escrow & Payout Engine
- Enforces automated two-phase settlement:
  1. **Phase 1 (Holding):** Buyer funds authorized and captured into platform escrow upon order placement.
  2. **Phase 2 (Release):** Upon physical quality verification at the Hub and delivery sign-off (via OTP confirmation), $P_{\text{farmer}}$ is directly settled to the farmer’s mobile financial account (MFS / Bank) minus zero or predetermined transparent platform fees.

---

## 5. Domain Data Models (Entities & Attributes)

### 5.1 Entity Specifications

#### Farmer Profile
- `farmer_id`: UUID (Primary Key)
- `phone_number`: String (Unique, primary auth identifier)
- `nid_or_tin`: String (National identification for KYC compliance)
- `hub_id`: UUID (Foreign Key $\rightarrow$ Assigned regional collection node)
- `geo_location`: `POINT(latitude, longitude)`
- `rating`: Float (Rolling composite of quality grading and fulfillment consistency)

#### Product / Listing
- `listing_id`: UUID (Primary Key)
- `farmer_id`: UUID (Foreign Key $\rightarrow$ Farmer Profile)
- `commodity_type`: Enum (`VEGETABLE`, `FRUIT`, `GRAIN`, `SPICE`, `DAIRY`)
- `sku_name`: String
- `grade`: Enum (`GRADE_A`, `GRADE_B`, `GRADE_C_BULK_PROCESS`)
- `available_qty`: Decimal (in standard base unit: kg)
- `unit_price`: Decimal (Base price per kg)
- `moq_retail`: Decimal (Minimum allowable purchase for B2C)
- `moq_wholesale`: Decimal (Minimum allowable purchase for B2B)
- `harvest_date`: Timestamp
- `expected_spoilage_window_hours`: Integer

#### Order & Order Items
- `order_id`: UUID (Primary Key)
- `buyer_id`: UUID (Foreign Key $\rightarrow$ Buyer Profile)
- `buyer_type`: Enum (`B2C_RETAIL`, `B2B_WHOLESALE`)
- `order_status`: Enum (`PENDING_CONFIRMATION`, `RESERVED`, `AGGREGATED_AT_HUB`, `IN_TRANSIT`, `DELIVERED`, `CANCELLED`)
- `subtotal_farmer_payout`: Decimal
- `service_fee`: Decimal
- `delivery_fee`: Decimal
- `total_payable`: Decimal
- `payment_status`: Enum (`ESCROW_HELD`, `SETTLED_TO_FARMER`, `REFUNDED`)

---

## 6. Non-Functional Requirements & Technical Constraints

### 6.1 Performance & Latency
- **API Latency:** $P_{95} \le 120\text{ ms}$ for search, catalog queries, and threshold validation.
- **Concurrent Checkout:** Support lock-free inventory deductions with Redis atomic decrements (`DECRBY`) backed by database writes via message queues.

### 6.2 Resiliency & Offline Operation
- **Low-Bandwidth Mobile Accessibility:** Farmer mobile client must operate seamlessly on unstable 2G/3G connections.
- **Offline Batch Sync:** Harvest submissions and local hub receipt scans can be recorded offline and synchronized idempotently via UUID reconciliation once connectivity is restored.

### 6.3 Security & Regulatory Standards
- **Strict Ledger Immutability:** Financial transaction records and payout deductions must follow double-entry bookkeeping semantics.
- **Traceability:** Every consumer batch maps back to `farmer_id`, `hub_id`, and `dispatch_timestamp` using QR-coded dispatch crates for contamination tracing and quality assurance.
