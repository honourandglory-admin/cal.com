# Cal.com Deployment Fix

## Problem
Deployment fails with error:
```
The pattern "app/api/cron/workflows/scheduleEmailReminders/route.ts" 
defined in functions doesn't match any Serverless Functions inside the api directory.
```

**Root cause:** The `functions` section in `apps/web/vercel.json` references paths like `app/api/...`, but with Root Directory set to `apps/web`, Vercel expects paths relative to that directory. These paths don't exist.

## Solution
Remove the `functions` section from `apps/web/vercel.json` (keeps cron jobs, which work fine). Function timeouts can be added later via Vercel dashboard if needed.

## Fix Prepared
I've created the fix on branch `fix/vercel-deployment` (commit `dc319d6`):
- Location: `/Users/mbp/clawd/projects/cal-com/` 
- Branch: `fix/vercel-deployment`
- Simplified `apps/web/vercel.json` to only include working crons section

## Options to Proceed

### Option 1: Grant chimbot-boop write access (fastest)
1. Go to https://github.com/honourandglory-admin/cal.com/settings/access
2. Add `chimbot-boop` as a collaborator with Write access
3. I'll push the fix branch and Vercel will auto-deploy

### Option 2: Jon manually applies the fix
1. Open `apps/web/vercel.json` in the honourandglory-admin/cal.com repo
2. Replace entire contents with:
```json
{
  "crons": [
    {
      "path": "/api/cron/calendar-subscriptions",
      "schedule": "*/5 * * * *"
    },
    {
      "path": "/api/cron/calendar-subscriptions-cleanup",
      "schedule": "0 3 * * *"
    },
    {
      "path": "/api/cron/queuedFormResponseCleanup",
      "schedule": "0 */12 * * *"
    },
    {
      "path": "/api/tasks/cron",
      "schedule": "* * * * *"
    },
    {
      "path": "/api/tasks/cleanup",
      "schedule": "0 0 * * *"
    },
    {
      "path": "/api/cron/credentials",
      "schedule": "*/5 * * * *"
    },
    {
      "path": "/api/cron/selected-calendars",
      "schedule": "*/5 * * * *"
    }
  ]
}
```
3. Commit and push
4. Vercel will auto-redeploy

### Option 3: Fork to chimbot-boop org
1. I fork the repo to chimbot-boop/cal.com
2. Deploy from there (requires new Vercel project)
3. More complex, but gives full control

## Recommendation
**Option 1** (grant write access) is fastest and cleanest - takes 30 seconds and unblocks all Cal.com work immediately.

## Impact
Once fixed, deployment should succeed in ~3-5 minutes. Then all 5 implementation phases (6.5h total work) can proceed.
