# LombaFest Database Schema

## Tables Overview

### 1. Users Table
Menyimpan informasi pengguna (peserta, organizer, admin)

```sql
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  email VARCHAR(255) UNIQUE NOT NULL,
  password VARCHAR(255) NOT NULL,
  name VARCHAR(255) NOT NULL,
  role VARCHAR(50) DEFAULT 'user',  -- user, organizer, admin
  avatar VARCHAR(500),
  phone VARCHAR(20),
  bio TEXT,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);
```

**Fields:**
- `id`: Primary key
- `email`: Email unik untuk login
- `password`: Password ter-hash dengan bcrypt
- `name`: Nama pengguna
- `role`: user (peserta), organizer (panitia), atau admin
- `avatar`: URL foto profil
- `phone`: Nomor telepon untuk kontak
- `bio`: Biografi atau deskripsi singkat

---

### 2. Events Table
Menyimpan informasi event/lomba

```sql
CREATE TABLE events (
  id SERIAL PRIMARY KEY,
  name VARCHAR(255) NOT NULL,
  description TEXT NOT NULL,
  category VARCHAR(100) NOT NULL,
  province VARCHAR(100) NOT NULL,
  city VARCHAR(100) NOT NULL,
  start_date DATE NOT NULL,
  end_date DATE NOT NULL,
  prize_pool BIGINT DEFAULT 0,
  image_url VARCHAR(500),
  organizer_id INTEGER REFERENCES users(id),
  status VARCHAR(50) DEFAULT 'pending',  -- pending, published, cancelled, completed
  views INTEGER DEFAULT 0,
  is_premium BOOLEAN DEFAULT FALSE,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);
```

**Fields:**
- `category`: Futsal, Sepak Bola, Bulutangkis, etc.
- `province`: Wilayah event (DKI Jakarta, Jawa Barat, etc.)
- `prize_pool`: Total hadiah dalam rupiah
- `organizer_id`: ID organizer/panitia
- `status`: Status publikasi event
- `is_premium`: Jika true, event ditampilkan di posisi premium
- `views`: Jumlah kali event dilihat

---

### 3. Registrations Table
Menyimpan data peserta yang mendaftar event

```sql
CREATE TABLE registrations (
  id SERIAL PRIMARY KEY,
  user_id INTEGER NOT NULL REFERENCES users(id),
  event_id INTEGER NOT NULL REFERENCES events(id),
  status VARCHAR(50) DEFAULT 'pending',  -- pending, confirmed, cancelled
  team_name VARCHAR(255),
  participants_count INTEGER DEFAULT 1,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);
```

**Fields:**
- `user_id`: ID peserta yang mendaftar
- `event_id`: ID event yang diikuti
- `team_name`: Nama tim (jika lomba tim)
- `participants_count`: Jumlah peserta dalam tim

---

### 4. Payments Table
Menyimpan data transaksi pembayaran

```sql
CREATE TABLE payments (
  id SERIAL PRIMARY KEY,
  user_id INTEGER NOT NULL REFERENCES users(id),
  event_id INTEGER REFERENCES events(id),
  registration_id INTEGER REFERENCES registrations(id),
  amount BIGINT NOT NULL,
  method VARCHAR(50) DEFAULT 'midtrans',  -- midtrans, transfer_bank, etc.
  transaction_id VARCHAR(255),
  status VARCHAR(50) DEFAULT 'pending',  -- pending, completed, failed, cancelled
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);
```

**Fields:**
- `amount`: Nominal pembayaran dalam rupiah
- `method`: Metode pembayaran (Midtrans, manual transfer, etc.)
- `transaction_id`: ID transaksi dari payment gateway
- `status`: Status pembayaran

---

### 5. Analytics Table
Menyimpan data statistik event harian

```sql
CREATE TABLE analytics (
  id SERIAL PRIMARY KEY,
  event_id INTEGER NOT NULL REFERENCES events(id),
  date DATE NOT NULL,
  views INTEGER DEFAULT 0,
  clicks INTEGER DEFAULT 0,
  registrations INTEGER DEFAULT 0,
  revenue BIGINT DEFAULT 0,
  created_at TIMESTAMP DEFAULT NOW()
);
```

**Fields:**
- `views`: Total views pada hari tersebut
- `clicks`: Total klik ke detail event
- `registrations`: Jumlah pendaftar baru
- `revenue`: Pendapatan dari event pada hari tersebut

---

## Relationships

```
Users (1) ──────────────── (Many) Events
  │                          │
  │                          │
  └─── (1)                   └─── (1)
       │                          │
       ├──────────────────────────┘
       │
  (Many) Registrations
       │
       ├────────────────────── (Many) Payments
```

---

## Indexes untuk Performance

```sql
-- Search indexes
CREATE INDEX idx_events_category ON events(category);
CREATE INDEX idx_events_province ON events(province);
CREATE INDEX idx_events_organizer ON events(organizer_id);
CREATE INDEX idx_events_status ON events(status);

-- Registration indexes
CREATE INDEX idx_registrations_user ON registrations(user_id);
CREATE INDEX idx_registrations_event ON registrations(event_id);

-- Payment indexes
CREATE INDEX idx_payments_user ON payments(user_id);
CREATE INDEX idx_payments_status ON payments(status);

-- Analytics indexes
CREATE INDEX idx_analytics_event_date ON analytics(event_id, date);
```

---

## Sample Queries

### Get Top Events by Views
```sql
SELECT id, name, views, prize_pool
FROM events
WHERE status = 'published'
ORDER BY views DESC
LIMIT 10;
```

### Get Revenue by Organizer
```sql
SELECT 
  u.id, u.name,
  COUNT(DISTINCT r.id) as total_registrations,
  COALESCE(SUM(p.amount), 0) as total_revenue
FROM users u
JOIN events e ON u.id = e.organizer_id
LEFT JOIN registrations r ON e.id = r.event_id
LEFT JOIN payments p ON r.id = p.registration_id AND p.status = 'completed'
WHERE u.role = 'organizer'
GROUP BY u.id, u.name
ORDER BY total_revenue DESC;
```

### Get Events by Province with Stats
```sql
SELECT 
  e.id, e.name, e.province,
  COUNT(DISTINCT r.id) as participant_count,
  COALESCE(SUM(p.amount), 0) as revenue
FROM events e
LEFT JOIN registrations r ON e.id = r.event_id
LEFT JOIN payments p ON r.id = p.registration_id AND p.status = 'completed'
WHERE e.status = 'published'
GROUP BY e.id, e.name, e.province
ORDER BY participant_count DESC;
```
