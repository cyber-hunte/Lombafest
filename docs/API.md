# LombaFest API Documentation

## Base URL
```
http://localhost:5000/api
```

## Authentication
Semua endpoint yang memerlukan autentikasi harus menyertakan header:
```
Authorization: Bearer <jwt_token>
```

---

## 📝 Auth Endpoints

### Register User
```http
POST /auth/register
Content-Type: application/json

{
  "email": "user@example.com",
  "password": "securepassword",
  "name": "John Doe",
  "role": "user" // user, organizer, admin
}

Response:
{
  "success": true,
  "user": {
    "id": 1,
    "email": "user@example.com",
    "name": "John Doe",
    "role": "user"
  },
  "token": "eyJhbGc..."
}
```

### Login
```http
POST /auth/login
Content-Type: application/json

{
  "email": "user@example.com",
  "password": "securepassword"
}

Response:
{
  "success": true,
  "user": { ... },
  "token": "eyJhbGc..."
}
```

---

## 🎯 Events Endpoints

### Get All Events
```http
GET /events?category=Futsal&province=DI%20Yogyakarta&search=cup

Query Parameters:
- category: string (optional) - Futsal, Sepak Bola, Bulutangkis, etc.
- province: string (optional) - DKI Jakarta, Jawa Barat, etc.
- search: string (optional) - Search by name or description
- limit: number (default: 100)
- offset: number (default: 0)

Response:
[
  {
    "id": 1,
    "name": "Lombafest Open Futsal Cup",
    "description": "...",
    "category": "Futsal",
    "province": "DI Yogyakarta",
    "city": "Yogyakarta",
    "start_date": "2024-07-06",
    "end_date": "2024-07-17",
    "prize_pool": 50000000,
    "image_url": "https://...",
    "organizer_id": 1,
    "status": "published",
    "views": 1250,
    "is_premium": false,
    "created_at": "2024-06-01T10:00:00Z"
  }
]
```

### Create Event (Organizer Only)
```http
POST /events
Authorization: Bearer <token>
Content-Type: application/json

{
  "name": "My Great Event",
  "description": "Event description",
  "category": "Futsal",
  "province": "DI Yogyakarta",
  "city": "Yogyakarta",
  "start_date": "2024-07-06",
  "end_date": "2024-07-17",
  "prize_pool": 50000000,
  "image_url": "https://..."
}

Response:
{
  "success": true,
  "event": { ... }
}
```

### Get Event by ID
```http
GET /events/:id

Response:
{
  "id": 1,
  "name": "Lombafest Open Futsal Cup",
  "description": "...",
  ...
}
```

### Update Event
```http
PUT /events/:id
Authorization: Bearer <token>
Content-Type: application/json

{ /* same fields as create */ }
```

### Delete Event
```http
DELETE /events/:id
Authorization: Bearer <token>
```

---

## 💳 Payment Endpoints

### Create Payment
```http
POST /payments/create
Authorization: Bearer <token>
Content-Type: application/json

{
  "event_id": 1,
  "amount": 100000,
  "payment_method": "midtrans"
}

Response:
{
  "success": true,
  "payment": {
    "id": 1,
    "user_id": 1,
    "event_id": 1,
    "amount": 100000,
    "status": "pending",
    "created_at": "2024-06-27T10:00:00Z"
  },
  "midtrans": {
    "token": "...",
    "redirect_url": "https://app.midtrans.com/snap/v3/..."
  }
}
```

### Payment Webhook (Midtrans)
```http
POST /payments/webhook
Content-Type: application/json

{
  "transaction_id": 1,
  "transaction_status": "capture"
}
```

---

## 👤 User Endpoints

### Get User Profile
```http
GET /users/profile
Authorization: Bearer <token>

Response:
{
  "id": 1,
  "email": "user@example.com",
  "name": "John Doe",
  "role": "user",
  "avatar": "https://...",
  "created_at": "2024-06-01T10:00:00Z"
}
```

### Get User Registrations
```http
GET /users/registrations
Authorization: Bearer <token>

Response:
[
  {
    "id": 1,
    "user_id": 1,
    "event_id": 1,
    "event_name": "Lombafest Open Futsal Cup",
    "status": "confirmed",
    "team_name": "My Team",
    "participants_count": 5,
    "created_at": "2024-06-27T10:00:00Z"
  }
]
```

---

## 📊 Analytics Endpoints

### Get Event Analytics
```http
GET /analytics/events/:eventId
Authorization: Bearer <token>

Response:
{
  "total_participants": 150,
  "paid_participants": 120,
  "total_revenue": 50000000,
  "views": 5000,
  "conversion_rate": 3.0
}
```

---

## 🛡️ Admin Endpoints

### Get Dashboard Stats
```http
GET /admin/dashboard
Authorization: Bearer <admin_token>

Response:
{
  "totalUsers": 5000,
  "totalEvents": 250,
  "totalRevenue": 500000000,
  "totalRegistrations": 15000
}
```

---

## Error Responses

### 400 Bad Request
```json
{
  "error": "Invalid input or missing fields"
}
```

### 401 Unauthorized
```json
{
  "error": "Access token required"
}
```

### 403 Forbidden
```json
{
  "error": "Insufficient permissions"
}
```

### 404 Not Found
```json
{
  "error": "Resource not found"
}
```

### 500 Server Error
```json
{
  "error": "Internal server error"
}
```
