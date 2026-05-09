# Zaidi Tailors — Complete Setup Guide
# ज़ैदी टेलर्स — सेटअप गाइड

---

## OPTION A: INSTANT USE (No setup needed)
Open `zaidi-tailors-app.html` in any mobile browser (Chrome/Safari).
Works completely offline. Data saved locally on device.
→ Share to phone: upload to Google Drive → open in Chrome mobile → "Add to Home Screen"

---

## OPTION B: Full Backend (PostgreSQL + Node.js + React)

### Prerequisites
- Node.js 18+
- PostgreSQL 14+
- npm

---

### 1. Database Setup

```sql
-- Create database
CREATE DATABASE zaidi_tailors;
\c zaidi_tailors

-- Inventory table (matches your AppSheet: item_id, item_brand, cloth_type, color, etc.)
CREATE TABLE cloth_inventory (
  id VARCHAR(20) PRIMARY KEY DEFAULT 'I' || substr(md5(random()::text), 1, 6),
  brand VARCHAR(100),
  cloth_type VARCHAR(100) NOT NULL,
  color VARCHAR(100),
  total_meters DECIMAL(10,2) NOT NULL,
  meters_available DECIMAL(10,2) NOT NULL,
  cost_per_meter DECIMAL(10,2) DEFAULT 0,
  sell_per_meter DECIMAL(10,2) DEFAULT 0,  -- Default: 1.6 × cost_per_meter (editable)
  date_added DATE DEFAULT CURRENT_DATE,
  notes TEXT,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

-- Product cloth config (Shirt=1.8m, Pant=1.3m, etc.)
CREATE TABLE product_config (
  id SERIAL PRIMARY KEY,
  product_name VARCHAR(100) UNIQUE NOT NULL,
  product_name_hi VARCHAR(100),
  meters_required DECIMAL(10,2) NOT NULL,
  updated_at TIMESTAMP DEFAULT NOW()
);

-- Customers
CREATE TABLE customers (
  id VARCHAR(20) PRIMARY KEY DEFAULT 'C' || substr(md5(random()::text), 1, 6),
  name VARCHAR(150) NOT NULL,
  phone VARCHAR(20),
  address TEXT,
  notes TEXT,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

-- Customer measurements (all fields from AppSheet)
CREATE TABLE customer_measurements (
  id VARCHAR(20) PRIMARY KEY DEFAULT 'M' || substr(md5(random()::text), 1, 6),
  customer_id VARCHAR(20) REFERENCES customers(id) ON DELETE CASCADE,
  label VARCHAR(100) DEFAULT 'Default',
  shoulder DECIMAL(5,2),
  chest DECIMAL(5,2),
  waist DECIMAL(5,2),
  hip DECIMAL(5,2),
  sleeve DECIMAL(5,2),
  length DECIMAL(5,2),
  collar DECIMAL(5,2),
  pant_length DECIMAL(5,2),
  thigh DECIMAL(5,2),
  bottom DECIMAL(5,2),
  notes TEXT,
  created_at TIMESTAMP DEFAULT NOW()
);

-- Orders (matches your AppSheet exactly)
CREATE TABLE orders (
  id VARCHAR(20) PRIMARY KEY DEFAULT substr(md5(random()::text), 1, 8),
  customer_id VARCHAR(20) REFERENCES customers(id),
  customer_name VARCHAR(150) NOT NULL,
  customer_phone VARCHAR(20),
  product VARCHAR(100) NOT NULL,
  cloth_source VARCHAR(20) DEFAULT 'shop' CHECK (cloth_source IN ('shop','customer')),
  inventory_id VARCHAR(20) REFERENCES cloth_inventory(id),
  meters_used DECIMAL(10,2),
  stitching_price DECIMAL(10,2) NOT NULL DEFAULT 0,
  material_cost DECIMAL(10,2) DEFAULT 0,   -- meters_used × sell_per_meter
  total_amount DECIMAL(10,2) DEFAULT 0,     -- stitching_price + material_cost
  advance_paid DECIMAL(10,2) DEFAULT 0,
  pending_amount DECIMAL(10,2) DEFAULT 0,   -- total_amount - advance_paid
  delivery_date DATE,
  status VARCHAR(30) DEFAULT 'pending'
    CHECK (status IN ('pending','in_progress','ready','delivered','cancelled')),
  notes TEXT,
  created_at DATE DEFAULT CURRENT_DATE,
  updated_at TIMESTAMP DEFAULT NOW()
);

-- Inventory usage log
CREATE TABLE inventory_log (
  id SERIAL PRIMARY KEY,
  inventory_id VARCHAR(20) REFERENCES cloth_inventory(id),
  order_id VARCHAR(20) REFERENCES orders(id),
  meters_deducted DECIMAL(10,2),
  created_at TIMESTAMP DEFAULT NOW()
);

-- Seed product config
INSERT INTO product_config (product_name, product_name_hi, meters_required) VALUES
  ('Shirt', 'कमीज', 1.8),
  ('Pant', 'पैंट', 1.3),
  ('Coat', 'कोट', 3.0),
  ('Suit', 'सूट', 4.5),
  ('Kurta', 'कुर्ता', 2.5),
  ('Salwar', 'सलवार', 2.0),
  ('Sherwani', 'शेरवानी', 5.0);
```

---

### 2. Backend Setup

```bash
cd backend
npm install
cp .env.example .env
# Edit .env: set DATABASE_URL=postgresql://user:pass@localhost:5432/zaidi_tailors
npm run dev
```

**Key Business Logic in Backend:**

```javascript
// Selling price = 1.6 × cost_per_meter (auto-calculated, editable)
const sell_per_meter = req.body.sell_per_meter || (cost_per_meter * 1.6);

// Material cost = meters_used × sell_per_meter  
const material_cost = meters_used * sell_per_meter;

// Total = stitching_price + material_cost
const total_amount = stitching_price + material_cost;

// Pending = total - advance_paid
const pending_amount = total_amount - advance_paid;

// Auto-deduct inventory on order creation
await db.query(
  'UPDATE cloth_inventory SET meters_available = meters_available - $1 WHERE id = $2',
  [meters_used, inventory_id]
);
```

---

### 3. Frontend Setup

```bash
cd frontend
npm install
npm start
# Open http://localhost:3000
```

---

### 4. Production Deployment

**Option 1: Railway (Recommended — free tier)**
```bash
# Install Railway CLI
npm install -g @railway/cli
railway login
railway init
railway up
# Railway auto-detects Node.js and PostgreSQL
```

**Option 2: Render.com**
1. Push to GitHub
2. Create Web Service → connect repo → set BUILD_COMMAND=`npm install` START_COMMAND=`node src/index.js`
3. Create PostgreSQL database → copy DATABASE_URL → add to env vars

**Option 3: VPS (DigitalOcean/Hetzner)**
```bash
# Install PM2
npm install -g pm2
cd backend && pm2 start src/index.js --name zaidi-backend
pm2 save && pm2 startup
```

---

## Business Logic Summary

| Calculation | Formula |
|-------------|---------|
| Sell Price/Meter | `Cost × 1.6` (default, editable) |
| Material Cost | `Meters Used × Sell Price/Meter` |
| Total Amount | `Stitching Price + Material Cost` |
| Pending Amount | `Total - Advance Paid` |
| Cloth Deduction | Auto on order creation (shop cloth only) |

---

## App Features

✅ Dashboard with today's stats  
✅ Orders with status tracking (Pending → In Progress → Ready → Delivered)  
✅ Inventory with brand, cloth type, color, meters tracking  
✅ Customer profiles with measurement records  
✅ 10 measurements per customer (shoulder, chest, waist, hip, sleeve, length, collar, pant length, thigh, bottom)  
✅ Auto material cost calculation  
✅ WhatsApp message generation (Hindi + English)  
✅ Voice input (Hindi + English)  
✅ Bilingual UI (English / Hindi toggle)  
✅ Sales & inventory reports  
✅ Low stock alerts  
✅ Works offline (localStorage version)  

---

## File Structure

```
zaidi-tailors/
├── zaidi-tailors-app.html     ← INSTANT USE (open in browser)
├── backend/
│   ├── src/
│   │   ├── index.js           ← Express server
│   │   ├── config/db.js       ← PostgreSQL pool
│   │   ├── config/migrate.js  ← Run schema migrations
│   │   ├── controllers/       ← Business logic
│   │   └── routes/            ← API endpoints
│   └── package.json
└── frontend/
    ├── src/
    │   ├── components/        ← React pages
    │   ├── i18n/en.json       ← English translations
    │   ├── i18n/hi.json       ← Hindi translations
    │   ├── contexts/          ← Language context
    │   └── hooks/useApi.js    ← API utilities
    └── package.json
```

---

*Zaidi Tailors App — Built for simplicity, runs anywhere*
