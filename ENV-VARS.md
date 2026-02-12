# Cal.com Environment Variables for Honour & Glory

## Secrets (Generated)
```bash
# NEXTAUTH_SECRET (for session encryption)
NEXTAUTH_SECRET=$(openssl rand -base64 32)

# CALENDSO_ENCRYPTION_KEY (for calendar credentials encryption)
CALENDSO_ENCRYPTION_KEY=$(openssl rand -hex 32)
```

Generated values:

```
NEXTAUTH_SECRET=wANgKbSMzF/QwPGyR2zGf3VmBqzml4BpjcdJAQrYb2Q=
CALENDSO_ENCRYPTION_KEY=14a76ac96b2a64b0d95dba311f13daf5ed1d99a1e59619cbc57e79a5973003f0
```

## Database (Jon needs to create in Vercel)
```bash
# Steps:
# 1. Go to https://vercel.com/honourandgloryboxingclub-9397/storage
# 2. Click "Create Database" → Select "Postgres"
# 3. Name it "cal-com-db" (region: London recommended)
# 4. Copy the connection string
# 5. Paste it for both DATABASE_URL and DATABASE_DIRECT_URL
```

## URLs (after deployment)
```bash
# Use custom domain or Vercel subdomain
NEXTAUTH_URL=https://book.honourandglory.co.uk
NEXT_PUBLIC_WEBAPP_URL=https://book.honourandglory.co.uk
NEXT_PUBLIC_WEBSITE_URL=https://book.honourandglory.co.uk
```

## Optional (can add later)
```bash
CALCOM_TELEMETRY_DISABLED=1
GOOGLE_LOGIN_ENABLED=true
# GOOGLE_API_CREDENTIALS= (for calendar sync - can configure later)
```

## Full Deployment Checklist

### 1. Create Postgres Database (Jon)
- [ ] Go to https://vercel.com/honourandgloryboxingclub-9397/storage
- [ ] Create Database → Postgres → Name: "cal-com-db" → Region: London
- [ ] Copy connection string

### 2. Import Repo to Vercel (Jon or Chimbot via CLI)
- [ ] Go to https://vercel.com/new/import
- [ ] Import honourandglory-admin/cal.com
- [ ] Or use CLI: `cd /Users/mbp/clawd/projects/cal-com && vercel`

### 3. Set Environment Variables
Paste all the above values into Vercel project settings → Environment Variables

### 4. Deploy
Vercel auto-deploys on import. Wait for build to complete.

### 5. Run Database Migrations
```bash
# In Vercel project settings → Functions → Terminal
npx prisma migrate deploy
```

### 6. Create Admin Account
Visit https://book.honourandglory.co.uk/auth/setup (or whatever domain Vercel assigns)

### 7. Configure Booking Types
- Trial session (60 mins, free/paid)
- Regular classes
- Private training sessions
