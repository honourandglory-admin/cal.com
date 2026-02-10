# Phase 4: Stripe Integration Implementation Plan

## Overview
Complete Stripe payment integration with webhook handling, payment tracking, and refund support for the H&G booking system.

**Estimated time:** 1.5 hours
**Dependencies:** Phase 3 (Kiosk UI) complete
**Tech stack:** Stripe API, Next.js API routes, Supabase

## Architecture

```
Kiosk UI → Payment Intent API → Stripe → Webhook → Supabase
                                            ↓
                                     Update booking
                                     Insert payment record
```

## Stripe Setup (20 mins)

### 1. Create Stripe Account
- Sign up at https://stripe.com (use boxing@agentmail.to)
- Enable test mode for development
- Production keys for live deployment

### 2. Store API Keys
Add to 1Password (chimbot-vault):
- Publishable key (starts with `pk_`)
- Secret key (starts with `sk_`)
- Webhook signing secret (starts with `whsec_`)

### 3. Configure Environment Variables
In Vercel project settings:

```bash
# Stripe keys
STRIPE_PUBLISHABLE_KEY=pk_live_...
STRIPE_SECRET_KEY=sk_live_...
STRIPE_WEBHOOK_SECRET=whsec_...

# Also add to .env.local for development
```

### 4. Install Stripe SDK
```bash
cd apps/web
npm install @stripe/stripe-js stripe
```

## Payment Intent Creation (30 mins)

### API Route: `/api/payments/create-intent`

**Purpose:** Server-side Payment Intent creation (secure, no key exposure)

```typescript
// apps/web/src/app/api/payments/create-intent/route.ts

import { NextRequest, NextResponse } from 'next/server';
import Stripe from 'stripe';
import { createClient } from '@supabase/supabase-js';

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!, {
  apiVersion: '2024-11-20.acacia',
});

const supabase = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY! // Server-side, can bypass RLS
);

export async function POST(request: NextRequest) {
  try {
    const { bookingId } = await request.json();
    
    // Fetch booking details
    const { data: booking, error } = await supabase
      .from('bookings')
      .select('*, class:classes(name), member:members(first_name, last_name, email)')
      .eq('id', bookingId)
      .single();
    
    if (error || !booking) {
      return NextResponse.json(
        { error: 'Booking not found' },
        { status: 404 }
      );
    }
    
    // Check if already paid
    if (booking.payment_status === 'paid') {
      return NextResponse.json(
        { error: 'Booking already paid' },
        { status: 400 }
      );
    }
    
    // Create Payment Intent
    const paymentIntent = await stripe.paymentIntents.create({
      amount: Math.round(booking.amount_gbp * 100), // Convert GBP to pence
      currency: 'gbp',
      metadata: {
        bookingId: booking.id,
        memberId: booking.member_id,
        className: booking.class.name,
        memberName: `${booking.member.first_name} ${booking.member.last_name}`,
      },
      receipt_email: booking.member.email || undefined,
      description: `H&G Boxing: ${booking.class.name} - ${booking.session_date}`,
    });
    
    // Update booking with Payment Intent ID
    await supabase
      .from('bookings')
      .update({ stripe_payment_id: paymentIntent.id })
      .eq('id', bookingId);
    
    return NextResponse.json({
      clientSecret: paymentIntent.client_secret,
      paymentIntentId: paymentIntent.id,
    });
    
  } catch (err: any) {
    console.error('Payment Intent creation failed:', err);
    return NextResponse.json(
      { error: err.message },
      { status: 500 }
    );
  }
}
```

### Client-Side Integration

Update `StripePayment.tsx` from Phase 3:

```typescript
import { loadStripe } from '@stripe/stripe-js';
import {
  Elements,
  PaymentElement,
  useStripe,
  useElements,
} from '@stripe/react-stripe-js';

const stripePromise = loadStripe(process.env.NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY!);

export default function StripePayment({ bookingId, amount, onSuccess, onError }) {
  const [clientSecret, setClientSecret] = useState<string | null>(null);
  
  useEffect(() => {
    // Create Payment Intent
    fetch('/api/payments/create-intent', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ bookingId }),
    })
      .then(res => res.json())
      .then(data => {
        if (data.error) throw new Error(data.error);
        setClientSecret(data.clientSecret);
      })
      .catch(err => onError(err.message));
  }, [bookingId]);
  
  if (!clientSecret) {
    return <div>Loading payment form...</div>;
  }
  
  return (
    <Elements stripe={stripePromise} options={{ clientSecret }}>
      <PaymentForm bookingId={bookingId} onSuccess={onSuccess} onError={onError} />
    </Elements>
  );
}

function PaymentForm({ bookingId, onSuccess, onError }) {
  const stripe = useStripe();
  const elements = useElements();
  const [isProcessing, setIsProcessing] = useState(false);
  
  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    
    if (!stripe || !elements) return;
    
    setIsProcessing(true);
    
    const { error, paymentIntent } = await stripe.confirmPayment({
      elements,
      confirmParams: {
        return_url: `${window.location.origin}/kiosk/success?bookingId=${bookingId}`,
      },
      redirect: 'if_required', // Stay on page if 3DS not needed
    });
    
    setIsProcessing(false);
    
    if (error) {
      onError(error.message);
    } else if (paymentIntent?.status === 'succeeded') {
      onSuccess(paymentIntent.id);
    }
  };
  
  return (
    <form onSubmit={handleSubmit}>
      <PaymentElement />
      <button
        type="submit"
        disabled={!stripe || isProcessing}
        className="w-full mt-4 bg-black text-white py-4 px-6 text-lg rounded-lg"
      >
        {isProcessing ? 'Processing...' : `Pay £${amount.toFixed(2)}`}
      </button>
    </form>
  );
}
```

## Webhook Handler (40 mins)

### API Route: `/api/webhooks/stripe`

**Purpose:** Listen for Stripe events and update database accordingly

```typescript
// apps/web/src/app/api/webhooks/stripe/route.ts

import { NextRequest, NextResponse } from 'next/server';
import Stripe from 'stripe';
import { createClient } from '@supabase/supabase-js';

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!, {
  apiVersion: '2024-11-20.acacia',
});

const supabase = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY!
);

// Disable body parsing - Stripe needs raw body for signature verification
export const config = {
  api: {
    bodyParser: false,
  },
};

export async function POST(request: NextRequest) {
  const body = await request.text();
  const signature = request.headers.get('stripe-signature');
  
  if (!signature) {
    return NextResponse.json(
      { error: 'No signature' },
      { status: 400 }
    );
  }
  
  let event: Stripe.Event;
  
  try {
    event = stripe.webhooks.constructEvent(
      body,
      signature,
      process.env.STRIPE_WEBHOOK_SECRET!
    );
  } catch (err: any) {
    console.error('Webhook signature verification failed:', err.message);
    return NextResponse.json(
      { error: `Webhook Error: ${err.message}` },
      { status: 400 }
    );
  }
  
  // Handle the event
  switch (event.type) {
    case 'payment_intent.succeeded':
      await handlePaymentSuccess(event.data.object as Stripe.PaymentIntent);
      break;
      
    case 'payment_intent.payment_failed':
      await handlePaymentFailure(event.data.object as Stripe.PaymentIntent);
      break;
      
    case 'charge.refunded':
      await handleRefund(event.data.object as Stripe.Charge);
      break;
      
    default:
      console.log(`Unhandled event type: ${event.type}`);
  }
  
  return NextResponse.json({ received: true });
}

async function handlePaymentSuccess(paymentIntent: Stripe.PaymentIntent) {
  const bookingId = paymentIntent.metadata.bookingId;
  
  if (!bookingId) {
    console.error('No bookingId in Payment Intent metadata');
    return;
  }
  
  // Update booking status
  await supabase
    .from('bookings')
    .update({
      payment_status: 'paid',
      status: 'confirmed',
    })
    .eq('id', bookingId);
  
  // Insert payment record
  await supabase
    .from('payments')
    .insert({
      member_id: paymentIntent.metadata.memberId,
      booking_id: bookingId,
      amount_gbp: paymentIntent.amount / 100,
      currency: paymentIntent.currency.toUpperCase(),
      payment_method: 'card',
      stripe_payment_intent_id: paymentIntent.id,
      stripe_status: paymentIntent.status,
      status: 'completed',
    });
  
  console.log(`Payment succeeded for booking ${bookingId}`);
}

async function handlePaymentFailure(paymentIntent: Stripe.PaymentIntent) {
  const bookingId = paymentIntent.metadata.bookingId;
  
  if (!bookingId) return;
  
  // Mark booking as payment failed
  await supabase
    .from('bookings')
    .update({
      payment_status: 'pending',
      status: 'cancelled',
    })
    .eq('id', bookingId);
  
  console.error(`Payment failed for booking ${bookingId}:`, paymentIntent.last_payment_error?.message);
}

async function handleRefund(charge: Stripe.Charge) {
  const paymentIntentId = charge.payment_intent as string;
  
  // Find payment record
  const { data: payment } = await supabase
    .from('payments')
    .select('*')
    .eq('stripe_payment_intent_id', paymentIntentId)
    .single();
  
  if (!payment) {
    console.error(`Payment not found for charge ${charge.id}`);
    return;
  }
  
  const refundAmount = charge.amount_refunded / 100;
  
  // Update payment record
  await supabase
    .from('payments')
    .update({
      status: 'refunded',
      refunded_at: new Date().toISOString(),
      refund_amount_gbp: refundAmount,
    })
    .eq('id', payment.id);
  
  // Update booking
  if (payment.booking_id) {
    await supabase
      .from('bookings')
      .update({
        payment_status: 'refunded',
        status: 'cancelled',
      })
      .eq('id', payment.booking_id);
  }
  
  console.log(`Refund processed for payment ${payment.id}: £${refundAmount}`);
}
```

### Stripe Dashboard Webhook Configuration

1. Go to Stripe Dashboard → Developers → Webhooks
2. Add endpoint: `https://your-domain.vercel.app/api/webhooks/stripe`
3. Select events to listen for:
   - `payment_intent.succeeded`
   - `payment_intent.payment_failed`
   - `charge.refunded`
4. Copy webhook signing secret → add to environment variables

## Cash Payment Handling (20 mins)

### API Route: `/api/payments/confirm-cash`

**Purpose:** Staff confirms cash payment received (admin dashboard action)

```typescript
// apps/web/src/app/api/payments/confirm-cash/route.ts

import { NextRequest, NextResponse } from 'next/server';
import { createClient } from '@supabase/supabase-js';

const supabase = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY!
);

export async function POST(request: NextRequest) {
  try {
    const { bookingId, staffUserId } = await request.json();
    
    // Fetch booking
    const { data: booking, error } = await supabase
      .from('bookings')
      .select('*')
      .eq('id', bookingId)
      .single();
    
    if (error || !booking) {
      return NextResponse.json(
        { error: 'Booking not found' },
        { status: 404 }
      );
    }
    
    // Update booking
    await supabase
      .from('bookings')
      .update({
        payment_status: 'paid',
        status: 'confirmed',
        checked_in_at: new Date().toISOString(),
      })
      .eq('id', bookingId);
    
    // Insert payment record
    await supabase
      .from('payments')
      .insert({
        member_id: booking.member_id,
        booking_id: bookingId,
        amount_gbp: booking.amount_gbp,
        currency: 'GBP',
        payment_method: 'cash',
        status: 'completed',
        notes: `Confirmed by staff user ${staffUserId}`,
      });
    
    return NextResponse.json({ success: true });
    
  } catch (err: any) {
    console.error('Cash payment confirmation failed:', err);
    return NextResponse.json(
      { error: err.message },
      { status: 500 }
    );
  }
}
```

## Testing (20 mins)

### Test Cases

**1. Successful card payment**
- Use Stripe test card: `4242 4242 4242 4242`
- Expiry: any future date
- CVC: any 3 digits
- Verify: booking status → confirmed, payment record created

**2. Failed payment (insufficient funds)**
- Use test card: `4000 0000 0000 9995`
- Verify: booking status → cancelled, no payment record

**3. 3D Secure authentication**
- Use test card: `4000 0025 0000 3155`
- Complete 3DS flow
- Verify: booking confirmed after authentication

**4. Cash payment**
- Create booking with payment_method='cash'
- Staff confirms via admin dashboard
- Verify: booking confirmed, payment record created

**5. Refund**
- Process successful payment
- Issue refund via Stripe Dashboard
- Verify: webhook updates booking to refunded, payment record updated

### Webhook Testing

Use Stripe CLI for local webhook testing:

```bash
# Install Stripe CLI
brew install stripe/stripe-cli/stripe

# Login
stripe login

# Forward webhooks to local server
stripe listen --forward-to localhost:3000/api/webhooks/stripe

# Trigger test events
stripe trigger payment_intent.succeeded
stripe trigger payment_intent.payment_failed
stripe trigger charge.refunded
```

## Error Handling

### Payment Intent Creation Errors
- Invalid booking ID → 404
- Already paid → 400
- Stripe API error → 500, log details

### Webhook Processing Errors
- Invalid signature → 400
- Missing metadata → log warning, skip processing
- Database update failure → retry with exponential backoff (Supabase functions)

### User-Facing Error Messages
- Generic: "Payment failed. Please try again or contact staff."
- Network error: "Connection lost. Please check your internet."
- Card declined: "Card declined. Please try another payment method."

## Security Checklist

- [ ] Stripe secret key stored in environment variables (never in code)
- [ ] Webhook signature verification enabled
- [ ] Payment Intent creation server-side only
- [ ] Supabase service role key used for admin operations
- [ ] RLS policies prevent unauthorized payment record access
- [ ] HTTPS enforced (Vercel handles this)
- [ ] Rate limiting on payment API routes (Vercel default: 100 req/10s)

## Phase 4 Implementation Checklist

### Stripe Setup (20 mins)
- [ ] Create Stripe account (boxing@agentmail.to)
- [ ] Store API keys in 1Password
- [ ] Configure environment variables in Vercel
- [ ] Install Stripe SDK (`@stripe/stripe-js`, `stripe`)

### Payment Intent API (30 mins)
- [ ] Create `/api/payments/create-intent` route
- [ ] Fetch booking details from Supabase
- [ ] Create Payment Intent with metadata
- [ ] Update booking with Payment Intent ID
- [ ] Update `StripePayment.tsx` client component
- [ ] Test end-to-end payment flow

### Webhook Handler (40 mins)
- [ ] Create `/api/webhooks/stripe` route
- [ ] Implement signature verification
- [ ] Handle `payment_intent.succeeded`
- [ ] Handle `payment_intent.payment_failed`
- [ ] Handle `charge.refunded`
- [ ] Configure webhook endpoint in Stripe Dashboard
- [ ] Test with Stripe CLI

### Cash Payment (20 mins)
- [ ] Create `/api/payments/confirm-cash` route
- [ ] Add staff confirmation UI (admin dashboard placeholder)
- [ ] Test cash payment flow

### Testing & Error Handling (20 mins)
- [ ] Test successful payment (card 4242...)
- [ ] Test failed payment (card 9995...)
- [ ] Test 3D Secure (card 3155...)
- [ ] Test cash confirmation
- [ ] Test refund webhook
- [ ] Verify error messages are user-friendly

## Next Steps (Phase 5)

Once Phase 4 complete, Phase 5 (Admin Dashboard) implements:
- Staff authentication
- Today's bookings list with real-time updates
- Cash payment confirmation UI (uses `/api/payments/confirm-cash`)
- Member management (CRUD operations)
- Payment history and reporting

**Total remaining after Phase 4:** ~1.5h for Phase 5
