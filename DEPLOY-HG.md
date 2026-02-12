# Cal.com Deployment for Honour & Glory

## Minimal Required Env Vars

### Critical (must set)
```
DATABASE_URL=            # Postgres connection string (Supabase or Vercel Postgres)
DATABASE_DIRECT_URL=     # Same as above (needed for migrations)
NEXTAUTH_SECRET=         # Generate: openssl rand -base64 32
NEXTAUTH_URL=            # e.g. https://book.honourandglory.co.uk
CALENDSO_ENCRYPTION_KEY= # Generate: openssl rand -hex 32
NEXT_PUBLIC_WEBAPP_URL=  # Same as NEXTAUTH_URL
NEXT_PUBLIC_WEBSITE_URL= # Same as NEXTAUTH_URL
```

### Optional but recommended
```
GOOGLE_API_CREDENTIALS=  # For Google Calendar sync
GOOGLE_LOGIN_ENABLED=true
NEXT_PUBLIC_EMBED_LIB_URL= # For embedding booking widget on H&G site
CALCOM_TELEMETRY_DISABLED=1
```

## Database Options

### Option A: Vercel Postgres (simplest, H&G already on Vercel Pro)
- Go to Vercel dashboard → Storage → Create Database → Postgres
- Copy connection string to DATABASE_URL and DATABASE_DIRECT_URL

### Option B: Supabase Free Tier
- Create project at supabase.com
- Copy connection string from Settings → Database

## Deployment Steps

1. Import honourandglory-admin/cal.com in Vercel dashboard
2. Set env vars (above)
3. Deploy (Vercel auto-detects Next.js)
4. Run DB migrations: `npx prisma migrate deploy`
5. Create admin account at /auth/setup
6. Configure booking types (trial session, etc.)

## Custom Domain
- Add `book.honourandglory.co.uk` (or similar) as custom domain in Vercel
- Update DNS: CNAME to cname.vercel-dns.com

## Integration with H&G Site
- Embed booking widget on trial page
- Or redirect /trial to Cal.com booking page
