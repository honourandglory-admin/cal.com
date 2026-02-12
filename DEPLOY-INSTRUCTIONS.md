# Cal.com Deployment Instructions for Jon

## What I've Done
1. ✅ Shallow cloned honourandglory-admin/cal.com (10,486 files)
2. ✅ Generated secrets (NEXTAUTH_SECRET + CALENDSO_ENCRYPTION_KEY)
3. ✅ Created env file template (`.env.deploy`)
4. ✅ Researched minimal required env vars (7 critical, 169 total)

## What You Need to Do in Vercel Dashboard

### Step 1: Create Postgres Database
1. Go to Vercel dashboard → honourandglory-admin account
2. Navigate to Storage → Create Database → Postgres
3. Name: `cal-hg-db` (or similar)
4. Copy the connection string (looks like `postgresql://username:password@host/dbname`)

### Step 2: Import Cal.com Repository
1. In Vercel dashboard → Add New Project
2. Import from GitHub: `honourandglory-admin/cal.com`
3. Framework Preset: Next.js (auto-detected)
4. Root Directory: `.` (leave default)

### Step 3: Set Environment Variables
Before deploying, add these env vars in the Vercel project settings:

```
DATABASE_URL=<paste_connection_string_from_step1>
DATABASE_DIRECT_URL=<same_as_above>
NEXTAUTH_SECRET=zxcL8tdOF5wp8X6ECO8M3bNHeUOsmjpPbd60vrrmOyo=
NEXTAUTH_URL=https://book.honourandglory.co.uk
CALENDSO_ENCRYPTION_KEY=65ba5c628f1efa2c1bca212a1a40f786c71d2ef7f6afd630f6584339d33d4c24
NEXT_PUBLIC_WEBAPP_URL=https://book.honourandglory.co.uk
NEXT_PUBLIC_WEBSITE_URL=https://book.honourandglory.co.uk
CALCOM_TELEMETRY_DISABLED=1
```

(Full list in `.env.deploy` file in this directory)

### Step 4: Deploy
1. Click "Deploy"
2. Vercel will build and deploy (takes ~3-5 mins)
3. You'll get a deployment URL like `cal-com-xyz.vercel.app`

### Step 5: Run Database Migrations
After first deployment succeeds:
1. In Vercel dashboard → Deployments → Latest deployment
2. Open the deployment URL in browser
3. You might see database errors - that's expected
4. Go to Vercel project → Settings → Functions
5. Add a new serverless function to run: `npx prisma migrate deploy`
   - Or SSH into Vercel deployment and run it manually
   - Or use Vercel CLI locally: `vercel env pull && npx prisma migrate deploy`

### Step 6: Set Up Admin Account
1. Navigate to `https://<your-deployment-url>/auth/setup`
2. Create your admin account
3. Configure initial booking event types (e.g., "Trial Session - 60 mins")

### Step 7: Add Custom Domain (Optional)
1. Vercel project → Settings → Domains
2. Add `book.honourandglory.co.uk`
3. Update DNS: CNAME → `cname.vercel-dns.com`
4. Update env vars to use new domain instead of `.vercel.app`

## Alternative: If You Want Me to Do It
If you can give me:
- Vercel account access (temp login or API token)
- Or add chimbot-boop as collaborator to honourandglory-admin Vercel team

Then I can complete the deployment automatically. Otherwise, the above steps are straightforward - should take ~15 mins total.

## After Deployment
Once it's live, we can:
- Embed booking widget on H&G trial page
- Or redirect `/trial` to Cal.com booking page
- Integrate with Google Calendar
- Build the member management layer (Phase 2)
