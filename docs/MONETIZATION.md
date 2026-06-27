# 💰 LombaFest Monetization Strategy

## Revenue Streams

### 1. Premium Event Listing
**Price:** Rp 100,000 - 500,000 per event
**Duration:** 7, 30, atau 90 hari
**Benefits:**
- Tampil di posisi teratas homepage
- Badge "Premium" atau "Featured"
- Prioritas di search results
- Email notification ke 10K+ subscribers
- Analytics premium (real-time, detail)

**Implementation:**
```javascript
// Event model
{
  is_premium: true,
  premium_until: "2024-08-27",
  premium_tier: "gold" // bronze, silver, gold
}

// Payment
router.post('/upgrade-premium', authenticateToken, async (req, res) => {
  const { eventId, tier, duration } = req.body; // tier: bronze|silver|gold, duration: 7|30|90
  
  const prices = {
    bronze: { 7: 100000, 30: 300000, 90: 800000 },
    silver: { 7: 250000, 30: 700000, 90: 1800000 },
    gold: { 7: 500000, 30: 1500000, 90: 4000000 }
  };
  
  const amount = prices[tier][duration];
  // Create payment...
});
```

---

### 2. Featured Homepage Position
**Price:** Rp 500,000 - 2,000,000 per minggu
**Slot:** Top 5 events on homepage
**Conversion:** ↑ 300-500% views
**Benefits:**
- Banner 300x250 atau 728x90
- Logo organizer
- Direct link to event
- 1 minggu full exposure

---

### 3. Sponsor Banner Ads
**Price:** Rp 2,000,000 - 10,000,000 per bulan
**Placement:**
- Homepage sidebar (2 slots)
- Event detail page (3 slots)
- Search results page
- Email blast footer

**Types:**
- Static Banner (300x250, 728x90, 970x250)
- Rotating Banner (3-5 images)
- Text Ads
- Video Ads (Rp 5M+)

**Implementation:**
```javascript
// Banner model
{
  id: 1,
  advertiser_id: 123,
  image_url: "https://...",
  destination_url: "https://...",
  placement: "homepage-sidebar", // homepage-sidebar, event-detail, search-results
  status: "active",
  start_date: "2024-06-27",
  end_date: "2024-07-27",
  daily_budget: 100000,
  clicks: 1250,
  impressions: 50000
}
```

---

### 4. Organizer Subscription
**Price:** Rp 49,000 - 499,000 per bulan

#### Tier 1: Basic (Rp 49K/bulan)
- Unlimited event creation
- Basic analytics
- Email support

#### Tier 2: Pro (Rp 199K/bulan)
- Everything in Basic
- Advanced analytics (real-time, conversion tracking)
- Email campaigns (unlimited)
- Priority support (24 jam)
- Custom domain subdomain (event.lombafest.id)

#### Tier 3: Enterprise (Rp 499K/bulan)
- Everything in Pro
- API access
- Custom integration
- Dedicated account manager
- White-label option
- Premium placement (top 10 events)

**Implementation:**
```javascript
router.post('/subscription/upgrade', authenticateToken, async (req, res) => {
  const { tier } = req.body; // basic, pro, enterprise
  const prices = { basic: 49000, pro: 199000, enterprise: 499000 };
  
  // Charge dengan Midtrans
  // Set subscription status
  await pool.query(
    'UPDATE users SET subscription_tier = $1, subscription_until = NOW() + INTERVAL \'1 month\' WHERE id = $2',
    [tier, req.user.userId]
  );
});
```

---

### 5. Commission from Ticketing
**Rate:** 10-15% per transaction
**When:** Jika platform handle ticketing/registration dengan fee

**Example:**
- Event charge: Rp 100,000 per peserta
- Platform fee: Rp 10,000 (10%)
- Organizer receive: Rp 90,000

**Implementation:**
```javascript
const COMMISSION_RATE = 0.10; // 10%

router.post('/registrations/:eventId', authenticateToken, async (req, res) => {
  const registrationFee = 100000;
  const platformCommission = registrationFee * COMMISSION_RATE; // 10000
  const organizerReceive = registrationFee - platformCommission; // 90000
  
  // Log transaction
  await pool.query(
    `INSERT INTO commissions (event_id, registration_id, gross_amount, commission, organizer_receive, created_at)
     VALUES ($1, $2, $3, $4, $5, NOW())`,
    [eventId, registrationId, registrationFee, platformCommission, organizerReceive]
  );
});
```

---

### 6. Email Blast Tool
**Model:** Pay-per-use
**Price:** Rp 5,000 - 50,000 per email campaign

**Tiers:**
- < 1,000 emails: Rp 5,000
- 1,000 - 10,000 emails: Rp 3,000/batch
- 10,000+ emails: Rp 2,000/batch (requires Pro subscription)

**Features:**
- Template builder
- Segmentation (category, province, age)
- A/B testing
- Open/click tracking
- Delivery reports

**Implementation:**
```javascript
router.post('/email-campaigns/send', authenticateToken, async (req, res) => {
  const { recipientCount, template } = req.body;
  const price = calculateEmailPrice(recipientCount);
  
  // Validate balance
  const user = await pool.query('SELECT wallet_balance FROM users WHERE id = $1', [req.user.userId]);
  
  if (user.rows[0].wallet_balance < price) {
    return res.status(400).json({ error: 'Insufficient balance' });
  }
  
  // Deduct from wallet
  await pool.query(
    'UPDATE users SET wallet_balance = wallet_balance - $1 WHERE id = $2',
    [price, req.user.userId]
  );
  
  // Send emails via Nodemailer
  // ...
});
```

---

### 7. API Access
**Price:** Rp 500,000 - 2,000,000 per bulan
**For:** Third-party app developers

**Endpoints Available:**
- Events API
- Registration API
- Analytics API
- Payment API

**Rate Limits:**
- Free: 100 requests/day
- Pro: 10,000 requests/day
- Enterprise: Unlimited

---

## Revenue Projections

### Year 1 (Conservative)
```
Premium Listings:      50 events × Rp 250K avg = Rp 12.5M
Featured Position:     20 slots × Rp 1M = Rp 20M
Sponsor Ads:          5 sponsors × Rp 5M = Rp 25M
Organizer Subs:       200 × Rp 150K avg = Rp 30M
Commission (1% GMV):  GMV Rp 1B × 1% = Rp 10M
                      ───────────────────
Total Year 1:         Rp 97.5M
```

### Year 2 (Growth)
```
20-30% growth in all streams
Total Year 2:         ~Rp 150M
```

### Year 3 (Maturity)
```
50%+ growth from mobile app + marketplace
Total Year 3:         ~Rp 300M+
```

---

## Payment Flow

```
Customer → Midtrans Gateway → Bank/e-wallet
   ↓                              ↓
   └──────────────────────────────┘
           Webhook Notification
           ↓
      Update Status
      Grant Access/Feature
      Send Confirmation Email
      Record Revenue
```

---

## Fraud Prevention

1. **Email Verification** - Confirm email before payment
2. **Phone Verification** - For organizers, verify phone
3. **Payment Verification** - Webhook dari payment gateway
4. **Chargeback Protection** - Document all transactions
5. **Duplicate Prevention** - Check duplicate registrations
6. **Rate Limiting** - Limit API calls per IP/user

---

## Dashboard Analytics

### For Admins
```
- Total Revenue (realtime)
- Top Events by Revenue
- Top Organizers
- Top Sponsors
- Growth Charts (weekly, monthly, yearly)
- Churn Rate
- Customer Lifetime Value (CLV)
```

### For Organizers
```
- Event Revenue
- Registrations per Day
- Conversion Rate
- Cost per Acquisition
- Premium ROI
- Email Campaign Performance
```

---

## Marketing Strategy

1. **Email Marketing** - Weekly newsletter ke 50K+ subscribers
2. **Influencer Partnerships** - 10-20% commission untuk affiliate
3. **Social Media** - Instagram, TikTok, YouTube campaigns
4. **SEO** - Optimize untuk long-tail keywords
5. **Paid Ads** - Google Ads, Facebook Ads untuk customer acquisition
6. **PR** - Press release untuk milestone (1 juta users, Rp 1M gross)
