# 🚀 LombaFest Deployment Guide

## Prerequisites

- Node.js 18+
- PostgreSQL 14+
- Docker & Docker Compose (optional)
- Vercel account (untuk frontend)
- Railway/Render account (untuk backend)

---

## Local Development

### 1. Setup Environment

```bash
git clone https://github.com/cyber-hunte/lombafest.git
cd lombafest
npm install

cp .env.example .env.local
```

### 2. Configure .env.local

```bash
# Database
DATABASE_URL=postgresql://lombafest:lombafest123@localhost:5432/lombafest
REDIS_URL=redis://localhost:6379

# JWT
JWT_SECRET=your-super-secret-key-change-this
JWT_EXPIRE=7d

# Midtrans
MIDTRANS_SERVER_KEY=<get-from-midtrans-dashboard>
MIDTRANS_CLIENT_KEY=<get-from-midtrans-dashboard>
MIDTRANS_ENV=sandbox

# OpenAI
OPENAI_API_KEY=sk-<your-key>

# Email
EMAIL_HOST=smtp.gmail.com
EMAIL_PORT=587
EMAIL_USER=your-email@gmail.com
EMAIL_PASS=<app-password>

NODE_ENV=development
NEXT_PUBLIC_API_URL=http://localhost:5000
```

### 3. Start with Docker Compose

```bash
docker-compose up -d
```

Ini akan start:
- PostgreSQL di port 5432
- Redis di port 6379

### 4. Run Migrations & Seed

```bash
npm run db:migrate
npm run db:seed
```

### 5. Start Development

```bash
npm run dev
```

- Frontend: http://localhost:3000
- API: http://localhost:5000
- Health check: http://localhost:5000/health

---

## Production Deployment

### Option A: Vercel + Railway

#### Frontend (Vercel)

1. **Push ke GitHub**
   ```bash
   git push origin main
   ```

2. **Deploy ke Vercel**
   - Buka https://vercel.com/new
   - Import GitHub repo
   - Set environment variables:
     - `NEXT_PUBLIC_API_URL`: https://api.lombafest.id
   - Deploy

3. **Domain Setup**
   - Beli domain di Namecheap/GoDaddy
   - Add nameservers di Vercel
   - Setup CNAME records

#### Backend (Railway)

1. **Prepare Railway**
   - Buka https://railway.app
   - Create new project
   - Add PostgreSQL plugin
   - Add Redis plugin

2. **Deploy from GitHub**
   - Connect GitHub repo
   - Set environment variables
   - Deploy

3. **Database Migration**
   ```bash
   # SSH ke Railway container
   railway shell
   npm run db:migrate
   ```

4. **Domain Setup**
   - Point custom domain ke Railway
   - Setup SSL certificate

---

### Option B: AWS EC2 + RDS

#### 1. Create EC2 Instance

```bash
# SSH ke instance
ssh -i key.pem ec2-user@your-instance-ip

# Update system
sudo yum update -y

# Install Node.js
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash
nvm install 18
nvm use 18

# Install PM2
npm install -g pm2

# Clone repo
git clone https://github.com/cyber-hunte/lombafest.git
cd lombafest
npm install
npm run build
```

#### 2. Setup RDS (PostgreSQL)

- Create RDS instance di AWS Console
- Configure security groups
- Get connection string
- Update DATABASE_URL di .env

#### 3. Start Application with PM2

```bash
# Create ecosystem config
cat > ecosystem.config.js << EOF
module.exports = {
  apps: [
    {
      name: 'lombafest-api',
      script: 'server/index.js',
      instances: 'max',
      exec_mode: 'cluster',
      env: {
        NODE_ENV: 'production',
        PORT: 5000
      }
    }
  ]
};
EOF

# Start
pm2 start ecosystem.config.js
pm2 save
pm2 startup
```

#### 4. Setup Nginx Reverse Proxy

```bash
# Install Nginx
sudo yum install -y nginx

# Configure
sudo cat > /etc/nginx/conf.d/lombafest.conf << EOF
upstream api {
  server localhost:5000;
}

server {
  listen 80;
  server_name api.lombafest.id;

  location / {
    proxy_pass http://api;
    proxy_set_header Host \$host;
    proxy_set_header X-Real-IP \$remote_addr;
    proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto \$scheme;
  }
}
EOF

# Start Nginx
sudo systemctl start nginx
sudo systemctl enable nginx
```

#### 5. SSL Certificate (Let's Encrypt)

```bash
sudo yum install -y certbot python3-certbot-nginx
sudo certbot certonly --nginx -d api.lombafest.id
```

---

## Docker Deployment

### 1. Build Image

```bash
docker build -t lombafest:latest .
```

### 2. Push to Docker Hub

```bash
docker tag lombafest:latest cyber-hunte/lombafest:latest
docker push cyber-hunte/lombafest:latest
```

### 3. Deploy with Docker Compose

```bash
cat > docker-compose.prod.yml << EOF
version: '3.8'
services:
  api:
    image: cyber-hunte/lombafest:latest
    ports:
      - '5000:5000'
    environment:
      DATABASE_URL: ${DATABASE_URL}
      REDIS_URL: ${REDIS_URL}
      NODE_ENV: production
    depends_on:
      - postgres
      - redis

  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: lombafest
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    volumes:
      - redis_data:/data

volumes:
  postgres_data:
  redis_data:
EOF

docker-compose -f docker-compose.prod.yml up -d
```

---

## Monitoring & Logging

### 1. PM2 Monitoring

```bash
pm2 monit
pm2 logs
pm2 logs --lines 100
```

### 2. Application Logs

```bash
# Log ke file
cat > server/index.js << EOF
const fs = require('fs');
const logStream = fs.createWriteStream('logs/app.log', { flags: 'a' });
console.log = function(msg) {
  logStream.write(new Date().toISOString() + ' - ' + msg + '\n');
};
EOF
```

### 3. Health Check

```bash
# Cek every 5 minutes
*/5 * * * * curl -f http://localhost:5000/health || systemctl restart lombafest
```

---

## Database Backup

```bash
# Backup
pg_dump $DATABASE_URL > backup-$(date +%Y%m%d-%H%M%S).sql

# Restore
psql $DATABASE_URL < backup.sql
```

---

## Performance Optimization

### 1. Database
```sql
-- Create indexes
CREATE INDEX idx_events_status ON events(status);
CREATE INDEX idx_registrations_event ON registrations(event_id);
CREATE INDEX idx_payments_status ON payments(status);

-- Enable query optimization
ANALYZE;
VACUUM;
```

### 2. Redis Caching
```javascript
// Cache popular events
const CACHE_TTL = 3600; // 1 hour
redis.setex('events:popular', CACHE_TTL, JSON.stringify(events));
```

### 3. CDN Setup
- Use CloudFlare atau AWS CloudFront
- Cache static assets (images, CSS, JS)
- Cache API responses untuk 5-10 menit

---

## Scaling Strategy

### Horizontal Scaling
1. Multiple API instances di load balancer
2. Read replicas untuk database
3. Redis cluster untuk caching

### Vertical Scaling
1. Upgrade instance size
2. Increase database resources
3. Optimize database queries

---

## Troubleshooting

### Port Already in Use
```bash
lsof -i :5000
kill -9 <PID>
```

### Database Connection Error
```bash
# Check connection
psql $DATABASE_URL

# Verify credentials
echo $DATABASE_URL
```

### Out of Memory
```bash
# Check memory
free -h
ps aux --sort=-%mem | head

# Restart app
pm2 restart all
```
