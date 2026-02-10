# Cal.com + Supabase Schema Design

## Overview
Custom booking and member management system for Honour & Glory Boxing Club, using Cal.com frontend + Supabase backend.

## Core Requirements
- Drop-in bookings (kiosk mode on tablet)
- Class attendance tracking
- Member profiles with contact info
- Payment tracking (Stripe integration)
- Admin dashboard for viewing bookings/members

## Database Schema

### `members`
Core member records. Each person who attends classes.

```sql
CREATE TABLE members (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW(),
  
  -- Identity
  first_name TEXT NOT NULL,
  last_name TEXT NOT NULL,
  email TEXT UNIQUE,
  phone TEXT,
  
  -- Demographics
  date_of_birth DATE,
  age_group TEXT CHECK (age_group IN ('infants', 'juniors', 'seniors')),
  
  -- Emergency contact
  emergency_contact_name TEXT,
  emergency_contact_phone TEXT,
  
  -- Medical
  medical_conditions TEXT,
  
  -- Membership status
  status TEXT DEFAULT 'active' CHECK (status IN ('active', 'inactive', 'suspended')),
  notes TEXT
);

CREATE INDEX idx_members_email ON members(email);
CREATE INDEX idx_members_status ON members(status);
```

### `classes`
Class definitions (not individual sessions - those are bookings).

```sql
CREATE TABLE classes (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  created_at TIMESTAMPTZ DEFAULT NOW(),
  
  -- Class identity
  name TEXT NOT NULL,
  slug TEXT UNIQUE NOT NULL,
  description TEXT,
  age_group TEXT CHECK (age_group IN ('infants', 'juniors', 'seniors')),
  
  -- Scheduling
  day_of_week INTEGER CHECK (day_of_week BETWEEN 0 AND 6), -- 0=Sunday
  start_time TIME NOT NULL,
  duration_minutes INTEGER NOT NULL,
  
  -- Capacity
  max_capacity INTEGER,
  
  -- Pricing
  drop_in_price_gbp NUMERIC(10,2),
  
  -- Status
  is_active BOOLEAN DEFAULT TRUE
);

CREATE INDEX idx_classes_active ON classes(is_active);
CREATE INDEX idx_classes_day_time ON classes(day_of_week, start_time);
```

### `bookings`
Individual session bookings. Links members to specific class sessions.

```sql
CREATE TABLE bookings (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  created_at TIMESTAMPTZ DEFAULT NOW(),
  
  -- Relationships
  member_id UUID NOT NULL REFERENCES members(id) ON DELETE CASCADE,
  class_id UUID NOT NULL REFERENCES classes(id) ON DELETE CASCADE,
  
  -- Session timing
  session_date DATE NOT NULL,
  session_start_time TIME NOT NULL,
  
  -- Booking metadata
  booking_type TEXT DEFAULT 'drop-in' CHECK (booking_type IN ('drop-in', 'membership', 'trial')),
  status TEXT DEFAULT 'confirmed' CHECK (status IN ('confirmed', 'cancelled', 'attended', 'no-show')),
  
  -- Payment
  payment_status TEXT DEFAULT 'pending' CHECK (payment_status IN ('pending', 'paid', 'refunded', 'waived')),
  amount_gbp NUMERIC(10,2),
  stripe_payment_id TEXT,
  
  -- Kiosk metadata
  checked_in_at TIMESTAMPTZ,
  created_via TEXT DEFAULT 'kiosk' CHECK (created_via IN ('kiosk', 'admin', 'online')),
  
  notes TEXT
);

CREATE INDEX idx_bookings_member ON bookings(member_id);
CREATE INDEX idx_bookings_class ON bookings(class_id);
CREATE INDEX idx_bookings_session ON bookings(session_date, session_start_time);
CREATE INDEX idx_bookings_status ON bookings(status);
CREATE INDEX idx_bookings_payment ON bookings(payment_status);
```

### `payments`
Payment transaction records (linked to Stripe).

```sql
CREATE TABLE payments (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  created_at TIMESTAMPTZ DEFAULT NOW(),
  
  -- Relationships
  member_id UUID NOT NULL REFERENCES members(id) ON DELETE CASCADE,
  booking_id UUID REFERENCES bookings(id) ON DELETE SET NULL,
  
  -- Payment details
  amount_gbp NUMERIC(10,2) NOT NULL,
  currency TEXT DEFAULT 'GBP',
  payment_method TEXT CHECK (payment_method IN ('card', 'cash', 'bank_transfer')),
  
  -- Stripe integration
  stripe_payment_intent_id TEXT UNIQUE,
  stripe_status TEXT,
  
  -- Metadata
  status TEXT DEFAULT 'pending' CHECK (status IN ('pending', 'completed', 'failed', 'refunded')),
  refunded_at TIMESTAMPTZ,
  refund_amount_gbp NUMERIC(10,2),
  notes TEXT
);

CREATE INDEX idx_payments_member ON payments(member_id);
CREATE INDEX idx_payments_stripe ON payments(stripe_payment_intent_id);
CREATE INDEX idx_payments_status ON payments(status);
```

### `admin_users`
Admin dashboard access control.

```sql
CREATE TABLE admin_users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW(),
  
  email TEXT UNIQUE NOT NULL,
  hashed_password TEXT, -- nullable if using OAuth only
  role TEXT DEFAULT 'staff' CHECK (role IN ('owner', 'staff', 'readonly')),
  is_active BOOLEAN DEFAULT TRUE,
  
  last_login_at TIMESTAMPTZ
);

CREATE INDEX idx_admin_users_email ON admin_users(email);
```

## Row Level Security (RLS) Policies

### Members table
```sql
ALTER TABLE members ENABLE ROW LEVEL SECURITY;

-- Kiosk: read-only for member lookup
CREATE POLICY "kiosk_read_members" ON members
  FOR SELECT
  USING (TRUE);

-- Admin: full access
CREATE POLICY "admin_all_members" ON members
  FOR ALL
  USING (auth.uid() IN (SELECT id FROM admin_users WHERE is_active = TRUE));
```

### Bookings table
```sql
ALTER TABLE bookings ENABLE ROW LEVEL SECURITY;

-- Kiosk: create + read own bookings
CREATE POLICY "kiosk_create_bookings" ON bookings
  FOR INSERT
  WITH CHECK (TRUE);

CREATE POLICY "kiosk_read_bookings" ON bookings
  FOR SELECT
  USING (TRUE);

-- Admin: full access
CREATE POLICY "admin_all_bookings" ON bookings
  FOR ALL
  USING (auth.uid() IN (SELECT id FROM admin_users WHERE is_active = TRUE));
```

### Payments table
```sql
ALTER TABLE payments ENABLE ROW LEVEL SECURITY;

-- No public access - admin only
CREATE POLICY "admin_all_payments" ON payments
  FOR ALL
  USING (auth.uid() IN (SELECT id FROM admin_users WHERE is_active = TRUE));
```

## Supabase Realtime Setup
Enable realtime for admin dashboard live updates:

```sql
ALTER PUBLICATION supabase_realtime ADD TABLE bookings;
ALTER PUBLICATION supabase_realtime ADD TABLE payments;
```

## Seed Data
Pre-populate classes from H&G site (Keystatic CMS):

```sql
-- Will be generated from src/content/classes/*.yaml during setup
-- Example:
INSERT INTO classes (name, slug, age_group, day_of_week, start_time, duration_minutes, drop_in_price_gbp, max_capacity)
VALUES
  ('Little Champs', 'little-champs', 'infants', 3, '16:00', 45, 8.00, 15),
  ('Junior Boxing', 'junior-boxing', 'juniors', 3, '17:00', 60, 10.00, 20),
  ('Adults Beginner', 'adults-beginner', 'seniors', 2, '19:00', 60, 12.00, 20);
```

## Implementation Notes

### Phase 2 Checklist:
- [ ] Create Supabase project (free tier sufficient for H&G)
- [ ] Run schema migration SQL
- [ ] Configure RLS policies
- [ ] Enable realtime for bookings/payments
- [ ] Generate service role key (for server-side operations)
- [ ] Generate anon key (for client-side kiosk)
- [ ] Store keys in 1Password (chimbot-vault)
- [ ] Seed classes from Keystatic

### Cal.com Integration Points:
- Cal.com handles the booking UI and calendar display
- On booking confirmation, Cal.com webhook triggers Supabase insert
- Supabase stores the actual booking records
- Admin dashboard reads directly from Supabase (bypasses Cal.com)

### Stripe Integration (Phase 4):
- Stripe Payment Intents API for card payments
- Webhooks to update payment_status in Supabase
- Support cash payments via admin dashboard (manual status update)

## Next Steps (Post-Deployment)
1. Create Supabase project via dashboard
2. Run schema SQL in SQL Editor
3. Configure environment variables in Vercel
4. Test kiosk booking flow
5. Build admin dashboard (Phase 5)
