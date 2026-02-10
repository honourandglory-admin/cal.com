# Phase 3: Kiosk Mode UI Implementation Plan

## Overview
Build a tablet-optimized drop-in booking interface for Honour & Glory Boxing Club. Guests walk in, select their class, book a session, and pay on the spot.

**Estimated time:** 2 hours
**Tech stack:** Cal.com frontend (Next.js/React), Supabase backend, Stripe Elements

## User Flow

### 1. Welcome Screen
**Purpose:** Immediate call-to-action for walk-ins

```
┌─────────────────────────────────────┐
│  🥊 HONOUR & GLORY BOXING CLUB      │
│                                      │
│  ┌──────────────────────────────┐  │
│  │                               │  │
│  │   BOOK YOUR DROP-IN SESSION   │  │
│  │                               │  │
│  │   [Large button, full width]  │  │
│  │                               │  │
│  └──────────────────────────────┘  │
│                                      │
│  Walk-ins welcome • Pay as you go   │
│  Classes from £8                    │
└─────────────────────────────────────┘
```

**Component:** `KioskWelcome.tsx`
- Large touch-friendly button (min 80px height)
- Auto-reset to welcome screen after 60s of inactivity
- Display current time and "Today's Classes" preview

### 2. Class Selection
**Purpose:** Choose which class to attend

```
┌─────────────────────────────────────┐
│  Select Your Class                   │
│                                      │
│  ┌────────────────────────────────┐ │
│  │ Little Champs (Ages 5-9)       │ │
│  │ Today 4:00 PM • 45 mins • £8   │ │
│  │ 12/15 spots available           │ │
│  └────────────────────────────────┘ │
│                                      │
│  ┌────────────────────────────────┐ │
│  │ Junior Boxing (Ages 10-16)     │ │
│  │ Today 5:00 PM • 60 mins • £10  │ │
│  │ 18/20 spots available           │ │
│  └────────────────────────────────┘ │
│                                      │
│  ┌────────────────────────────────┐ │
│  │ Adults Beginner                │ │
│  │ Today 7:00 PM • 60 mins • £12  │ │
│  │ 15/20 spots available           │ │
│  └────────────────────────────────┘ │
│                                      │
│  [← Back]                            │
└─────────────────────────────────────┘
```

**Component:** `ClassSelection.tsx`
- Query Supabase `classes` table filtered by `day_of_week` = today
- Show only upcoming classes (start_time > now + 15 mins buffer)
- Display real-time capacity from `bookings` count
- Disable if class is full
- Sort by start_time ascending

**Data fetch:**
```typescript
const { data: classes } = await supabase
  .from('classes')
  .select('*, bookings(count)')
  .eq('is_active', true)
  .eq('day_of_week', new Date().getDay())
  .gte('start_time', addMinutes(new Date(), 15))
  .order('start_time', { ascending: true });
```

### 3. Member Lookup / New Member
**Purpose:** Identify the person booking (returning member or new guest)

```
┌─────────────────────────────────────┐
│  Who's attending?                    │
│                                      │
│  ┌────────────────────────────────┐ │
│  │ Search by phone or name         │ │
│  │ [Text input - large keyboard]   │ │
│  └────────────────────────────────┘ │
│                                      │
│  Recent members:                     │
│  ┌────────────────────────────────┐ │
│  │ Sarah Johnson (07712 345678)   │ │
│  └────────────────────────────────┘ │
│  ┌────────────────────────────────┐ │
│  │ Mike Peters (07798 123456)     │ │
│  └────────────────────────────────┘ │
│                                      │
│  ┌────────────────────────────────┐ │
│  │ + I'm a new member             │ │
│  └────────────────────────────────┘ │
│                                      │
│  [← Back]                            │
└─────────────────────────────────────┘
```

**Component:** `MemberLookup.tsx`
- Debounced search on `members` table (phone, first_name, last_name)
- Show last 5 members who booked in past 7 days as quick-select
- "New member" button leads to registration form

**New member form (`NewMemberForm.tsx`):**
- First name, last name (required)
- Phone number (required, used as lookup key)
- Email (optional but encouraged)
- Date of birth (required for age verification)
- Emergency contact name + phone (required for minors)
- Medical conditions (optional textarea)
- Terms acceptance checkbox

### 4. Booking Confirmation
**Purpose:** Review details before payment

```
┌─────────────────────────────────────┐
│  Confirm Booking                     │
│                                      │
│  Member: Sarah Johnson               │
│  Class: Junior Boxing                │
│  When: Today, 5:00 PM (60 mins)      │
│  Cost: £10.00                        │
│                                      │
│  ┌────────────────────────────────┐ │
│  │ Pay £10.00 with card           │ │
│  └────────────────────────────────┘ │
│                                      │
│  ┌────────────────────────────────┐ │
│  │ Pay £10.00 cash                │ │
│  │ (confirm with staff)            │ │
│  └────────────────────────────────┘ │
│                                      │
│  [← Back]                            │
└─────────────────────────────────────┘
```

**Component:** `BookingConfirmation.tsx`
- Display summary of selection
- Two payment options: card (Stripe) or cash (manual confirmation)
- Insert booking record in Supabase with `payment_status: 'pending'`

### 5a. Card Payment (Stripe)
**Purpose:** Secure payment processing

```
┌─────────────────────────────────────┐
│  Pay £10.00                          │
│                                      │
│  [Stripe Card Element]               │
│                                      │
│  ┌────────────────────────────────┐ │
│  │ Complete Payment               │ │
│  └────────────────────────────────┘ │
│                                      │
│  [← Back]                            │
└─────────────────────────────────────┘
```

**Component:** `StripePayment.tsx`
- Stripe Payment Element (handles card input + validation)
- Create Payment Intent server-side (Next.js API route)
- On success: update booking `payment_status: 'paid'`, insert payment record
- Redirect to success screen

**API route:** `/api/create-payment-intent`
```typescript
export async function POST(req: Request) {
  const { bookingId, amount } = await req.json();
  
  const paymentIntent = await stripe.paymentIntents.create({
    amount: amount * 100, // Convert GBP to pence
    currency: 'gbp',
    metadata: { bookingId },
  });
  
  return Response.json({ clientSecret: paymentIntent.client_secret });
}
```

### 5b. Cash Payment
**Purpose:** Staff-confirmed manual payment

```
┌─────────────────────────────────────┐
│  Cash Payment                        │
│                                      │
│  Please pay £10.00 to staff          │
│                                      │
│  [Staff will confirm payment]        │
│                                      │
│  Waiting for confirmation...         │
│  ⏳                                  │
│                                      │
└─────────────────────────────────────┘
```

**Component:** `CashPaymentWaiting.tsx`
- Insert booking with `payment_status: 'pending'`, `payment_method: 'cash'`
- Show QR code or booking ID for staff to confirm
- Poll booking status every 3 seconds
- Staff uses admin dashboard to mark payment received
- Auto-timeout after 5 minutes → cancel booking

### 6. Success Screen
**Purpose:** Confirmation and next steps

```
┌─────────────────────────────────────┐
│  ✅ Booking Confirmed!               │
│                                      │
│  Sarah Johnson                       │
│  Junior Boxing                       │
│  Today at 5:00 PM                    │
│                                      │
│  Please arrive 10 minutes early      │
│  Bring water and indoor trainers     │
│                                      │
│  [Print Receipt] [Email Receipt]     │
│                                      │
│  Returning to welcome screen in 10s  │
└─────────────────────────────────────┘
```

**Component:** `BookingSuccess.tsx`
- Display booking details
- Update booking `status: 'confirmed'`
- Optional: send confirmation email (via Resend/SendGrid)
- Auto-redirect to welcome screen after 10 seconds

## Technical Implementation

### File Structure
```
apps/web/src/kiosk/
├── components/
│   ├── KioskWelcome.tsx
│   ├── ClassSelection.tsx
│   ├── MemberLookup.tsx
│   ├── NewMemberForm.tsx
│   ├── BookingConfirmation.tsx
│   ├── StripePayment.tsx
│   ├── CashPaymentWaiting.tsx
│   └── BookingSuccess.tsx
├── hooks/
│   ├── useKioskSession.ts      // Auto-reset timer
│   ├── useClassAvailability.ts // Real-time capacity
│   └── useMemberSearch.ts      // Debounced search
├── lib/
│   ├── supabase-client.ts      // Supabase anon client
│   └── kiosk-utils.ts          // Helpers
└── pages/
    └── kiosk.tsx               // Main kiosk route
```

### State Management
Use React Context + useState for kiosk session:

```typescript
type KioskState = {
  step: 'welcome' | 'classes' | 'member' | 'confirm' | 'payment' | 'success';
  selectedClass?: Class;
  selectedMember?: Member;
  bookingId?: string;
  resetTimer: () => void;
};
```

### Auto-Reset Timer
```typescript
function useKioskSession() {
  const [lastActivity, setLastActivity] = useState(Date.now());
  
  useEffect(() => {
    const timer = setInterval(() => {
      if (Date.now() - lastActivity > 60000) {
        resetToWelcome();
      }
    }, 1000);
    return () => clearInterval(timer);
  }, [lastActivity]);
  
  const resetTimer = () => setLastActivity(Date.now());
  
  return { resetTimer };
}
```

### Styling
- Tailwind CSS with large touch targets (min 60px)
- Font size: min 18px for body, 24px+ for headings
- High contrast (WCAG AA)
- No hover states (tablet has no mouse)
- Large tap areas with padding

### Supabase RLS
Already configured in SUPABASE-SCHEMA.md:
- Kiosk uses `anon` key (read members, insert bookings)
- No write access to payments table (only via webhook)

## Phase 3 Checklist

### Setup (15 mins)
- [ ] Create `/apps/web/src/kiosk/` directory structure
- [ ] Configure Supabase client with anon key
- [ ] Add Stripe publishable key to env vars
- [ ] Create `/pages/kiosk.tsx` route

### Components (90 mins)
- [ ] Build KioskWelcome (15 mins)
- [ ] Build ClassSelection with capacity query (20 mins)
- [ ] Build MemberLookup + search (20 mins)
- [ ] Build NewMemberForm with validation (20 mins)
- [ ] Build BookingConfirmation (10 mins)
- [ ] Build StripePayment integration (15 mins)
- [ ] Build CashPaymentWaiting with polling (15 mins)
- [ ] Build BookingSuccess with auto-reset (10 mins)

### Hooks & Utils (15 mins)
- [ ] useKioskSession with auto-reset timer
- [ ] useClassAvailability real-time subscription
- [ ] useMemberSearch debounced

### Testing (30 mins)
- [ ] End-to-end booking flow (card payment)
- [ ] End-to-end booking flow (cash payment)
- [ ] Member search and new member registration
- [ ] Auto-reset after inactivity
- [ ] Capacity enforcement (full class prevention)
- [ ] Edge cases: payment failure, network loss, timeout

### Stripe Integration Notes
- Use Stripe Payment Element (not legacy Card Element)
- Create Payment Intent server-side for security
- Webhook handler for `payment_intent.succeeded` to update booking status
- Handle 3D Secure redirects
- Store `stripe_payment_intent_id` in bookings table

### Next Steps (Phase 4)
Once Phase 3 complete, Phase 4 (Stripe webhook handler + payment tracking) can begin.

## Deployment
- Deploy to Vercel (same project as Cal.com)
- Access via `/kiosk` route
- Configure tablet to open in kiosk mode (fullscreen, no browser chrome)
- iPad/Android tablet recommended (min 10" screen)

## Future Enhancements (Post-MVP)
- QR code check-in for confirmed bookings
- Membership package purchase (10-class pass, monthly)
- Multi-language support (if needed for diverse community)
- Guest waiver signing (digital signature)
- Photo capture for member profiles
