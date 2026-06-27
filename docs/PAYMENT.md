# 💳 Payment Integration Guide (Midtrans)

## Overview

LombaFest menggunakan **Midtrans Snap** untuk payment gateway yang mendukung:
- Credit Card (Visa, Mastercard, American Express)
- Debit Card
- E-wallet (GCash, OVO, Dana, LinkAja, Sakuku)
- Bank Transfer (Virtual Account)
- Convenience Store (Indomaret, Alfamart)
- Buy Now Pay Later (Kredivo, Akulaku)

---

## Setup Midtrans Account

### 1. Register
- Buka https://dashboard.midtrans.com
- Daftar dengan email perusahaan
- Verifikasi email
- Lengkapi data company

### 2. Get API Keys
- Login ke Midtrans Dashboard
- Go to Settings → Access Keys
- Copy **Server Key** dan **Client Key**
- Ada 2 environment: Sandbox (testing) & Production

### 3. Configure in LombaFest

```env
# .env.local
MIDTRANS_SERVER_KEY=VT02-xxxxx (dari dashboard)
MIDTRANS_CLIENT_KEY=VT02-xxxxx (dari dashboard)
MIDTRANS_ENV=sandbox # atau production
```

---

## Payment Flow

```
┌─────────┐
│  User   │
└────┬────┘
     │ 1. Click "Register Event"
     ↓
┌──────────────────┐
│ Create Payment   │ POST /payments/create
│ in LombaFest     │
└────┬─────────────┘
     │ 2. Create Midtrans Transaction
     ↓
┌──────────────────┐
│  Midtrans API    │ Return: token + redirect_url
│  (Server-side)   │
└────┬─────────────┘
     │ 3. Redirect to Midtrans Snap
     ↓
┌──────────────────┐
│ Midtrans Snap    │ Payment UI
│ Payment Page     │ (iframe/popup)
└────┬─────────────┘
     │ 4. User input payment method
     │    & confirm payment
     ↓
┌──────────────────┐
│  Payment Gateway │ Process payment
│  (Bank/e-wallet) │
└────┬─────────────┘
     │ 5. Success/Failed
     ↓
┌──────────────────┐
│ Midtrans Webhook │ POST /payments/webhook
│ Notification     │ Update payment status
└────┬─────────────┘
     │ 6. Update LombaFest DB
     ↓
┌──────────────────┐
│ User Confirmed   │ Registration complete
│ for Event        │ Send confirmation email
└──────────────────┘
```

---

## Implementation

### Frontend (Next.js)

#### 1. Create Payment

```typescript
// app/events/[id]/register/page.tsx
'use client';

import axios from 'axios';
import { useRouter } from 'next/navigation';
const router = useRouter();

const handleRegister = async () => {
  try {
    const response = await axios.post('/api/payments/create', {
      event_id: eventId,
      amount: 100000,
      payment_method: 'midtrans'
    }, {
      headers: {
        'Authorization': `Bearer ${token}`
      }
    });

    const { midtrans } = response.data;
    
    // Redirect ke Midtrans Snap
    window.location.href = midtrans.redirect_url;
    
  } catch (error) {
    console.error('Payment error:', error);
  }
};

return (
  <button onClick={handleRegister}>
    Daftar Event (Rp 100.000)
  </button>
);
```

#### 2. Handle Callback

```typescript
// app/payments/callback/page.tsx
'use client';

import { useEffect } from 'react';
import { useRouter } from 'next/navigation';

export default function PaymentCallback() {
  const router = useRouter();
  const searchParams = new URLSearchParams(window.location.search);
  const orderId = searchParams.get('order_id');

  useEffect(() => {
    const checkPaymentStatus = async () => {
      const response = await fetch(`/api/payments/status?order_id=${orderId}`);
      const data = await response.json();

      if (data.status === 'completed') {
        router.push('/dashboard/my-events?success=true');
      } else if (data.status === 'failed') {
        router.push('/dashboard/my-events?error=payment_failed');
      }
    };

    checkPaymentStatus();
  }, [orderId, router]);

  return <div>Processing payment...</div>;
}
```

### Backend (Express)

#### 1. Create Payment Transaction

```javascript
// server/routes/payments.js
const midtransClient = require('midtrans-client');

const snap = new midtransClient.Snap({
  isProduction: process.env.MIDTRANS_ENV === 'production',
  serverKey: process.env.MIDTRANS_SERVER_KEY,
  clientKey: process.env.MIDTRANS_CLIENT_KEY,
});

router.post('/create', authenticateToken, async (req, res) => {
  try {
    const { event_id, amount, payment_method } = req.body;
    const user_id = req.user.userId;

    // Get user email
    const userResult = await pool.query(
      'SELECT email, name FROM users WHERE id = $1',
      [user_id]
    );
    const { email, name } = userResult.rows[0];

    // Create payment record
    const paymentResult = await pool.query(
      `INSERT INTO payments (user_id, event_id, amount, method, status, created_at)
       VALUES ($1, $2, $3, $4, 'pending', NOW())
       RETURNING id`,
      [user_id, event_id, amount, payment_method]
    );
    const paymentId = paymentResult.rows[0].id;

    // Create Midtrans transaction
    const parameter = {
      transaction_details: {
        order_id: `LF-${paymentId}-${Date.now()}`,
        gross_amount: amount,
      },
      customer_details: {
        first_name: name,
        email: email,
        phone: '+62',
      },
      callbacks: {
        finish: `${process.env.NEXT_PUBLIC_API_URL}/payments/callback`,
        unfinish: `${process.env.NEXT_PUBLIC_API_URL}/payments/callback`,
        error: `${process.env.NEXT_PUBLIC_API_URL}/payments/callback`,
      },
    };

    const transaction = await snap.createTransaction(parameter);

    res.json({
      success: true,
      payment: {
        id: paymentId,
        order_id: parameter.transaction_details.order_id,
        amount: amount,
        status: 'pending',
      },
      midtrans: {
        token: transaction.token,
        redirect_url: transaction.redirect_url,
      },
    });
  } catch (error) {
    console.error('Payment creation error:', error);
    res.status(400).json({ error: error.message });
  }
});
```

#### 2. Webhook Handler

```javascript
// server/routes/payments.js
const crypto = require('crypto');

router.post('/webhook', express.raw({type: 'application/json'}), async (req, res) => {
  try {
    const rawBody = req.body.toString('utf-8');
    const body = JSON.parse(rawBody);
    
    // Verify signature
    const signature = req.headers['x-signature'];
    const expectedSignature = crypto
      .createHash('sha512')
      .update(body.order_id + body.status_code + body.gross_amount + process.env.MIDTRANS_SERVER_KEY)
      .digest('hex');

    if (signature !== expectedSignature) {
      return res.status(403).json({ error: 'Invalid signature' });
    }

    const { order_id, transaction_status, payment_type } = body;
    
    // Extract payment ID from order_id
    const paymentId = parseInt(order_id.split('-')[1]);

    // Update payment status
    let status = 'pending';
    if (transaction_status === 'capture' || transaction_status === 'settlement') {
      status = 'completed';
      // Create registration record
      const paymentData = await pool.query(
        'SELECT * FROM payments WHERE id = $1',
        [paymentId]
      );
      const { user_id, event_id } = paymentData.rows[0];
      
      await pool.query(
        `INSERT INTO registrations (user_id, event_id, status, created_at)
         VALUES ($1, $2, 'confirmed', NOW())`,
        [user_id, event_id]
      );
      
      // Send confirmation email
      await sendConfirmationEmail(user_id, event_id);
      
    } else if (transaction_status === 'deny' || transaction_status === 'cancel') {
      status = 'failed';
    } else if (transaction_status === 'pending') {
      status = 'pending';
    } else if (transaction_status === 'expire') {
      status = 'expired';
    }

    await pool.query(
      'UPDATE payments SET status = $1, updated_at = NOW() WHERE id = $2',
      [status, paymentId]
    );

    res.json({ success: true });
  } catch (error) {
    console.error('Webhook error:', error);
    res.status(500).json({ error: error.message });
  }
});
```

#### 3. Get Payment Status

```javascript
router.get('/status', async (req, res) => {
  try {
    const { order_id } = req.query;

    const transaction = await snap.transaction.status(order_id);

    let status = 'pending';
    if (transaction.transaction_status === 'capture' || transaction.transaction_status === 'settlement') {
      status = 'completed';
    } else if (transaction.transaction_status === 'deny' || transaction.transaction_status === 'cancel') {
      status = 'failed';
    }

    res.json({ status, transaction });
  } catch (error) {
    console.error('Status check error:', error);
    res.status(400).json({ error: error.message });
  }
});
```

---

## Testing (Sandbox Mode)

### Test Credit Card Numbers

```
Visa:
4111111111111111
Expiry: 12/25
CVV: 123

Mastercard:
5264221856435416
Expiry: 12/25
CVV: 123

BCA Virtual Account:
1234567890
```

### Test Webhook

```bash
curl -X POST http://localhost:5000/api/payments/webhook \
  -H "Content-Type: application/json" \
  -d '{
    "order_id": "LF-123-456",
    "status_code": "200",
    "gross_amount": "100000",
    "transaction_status": "settlement",
    "payment_type": "credit_card"
  }'
```

---

## Settlement & Payouts

### 1. Settlement Schedule
- T+1: Dana tersedia di Midtrans account
- T+2-5: Dana transfer ke rekening bank organizer
- Biaya admin: 1.5% - 2.9% (tergantung payment method)

### 2. Payout Setup

```javascript
// Organizer setup rekening bank untuk payout
router.post('/organizer/bank-account', authenticateToken, async (req, res) => {
  const { bank_code, account_number, account_name } = req.body;
  
  await pool.query(
    `UPDATE users SET 
      bank_code = $1, 
      bank_account = $2, 
      bank_account_name = $3 
     WHERE id = $4`,
    [bank_code, account_number, account_name, req.user.userId]
  );
  
  res.json({ success: true });
});
```

### 3. Revenue Dashboard

```javascript
// Get organizer earnings
router.get('/earnings', authenticateToken, async (req, res) => {
  const result = await pool.query(
    `SELECT 
      COALESCE(SUM(amount), 0) as total_revenue,
      COALESCE(SUM(amount * 0.025), 0) as admin_fee,
      COALESCE(SUM(amount * 0.975), 0) as organizer_earning
     FROM payments p
     JOIN events e ON p.event_id = e.id
     WHERE e.organizer_id = $1 AND p.status = 'completed'`,
    [req.user.userId]
  );
  
  res.json(result.rows[0]);
});
```

---

## Troubleshooting

### Payment Failed
1. Check Midtrans Dashboard → Transactions
2. Verify amount, user email, order ID
3. Check server logs untuk error message
4. Test dengan sandbox keys dulu

### Webhook Not Received
1. Verify webhook URL di settings
2. Check firewall/port blocking
3. Use ngrok untuk test locally:
   ```bash
   ngrok http 5000
   Update NEXT_PUBLIC_API_URL di Midtrans
   ```

### Settlement Issue
1. Verify bank account di organizer profile
2. Check dengan Midtrans support
3. Ensure KYC verification complete
