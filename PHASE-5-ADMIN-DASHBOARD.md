# Phase 5: Admin Dashboard Implementation Plan

## Overview
Build a staff-facing admin dashboard for managing bookings, confirming cash payments, viewing member profiles, and tracking revenue.

**Estimated time:** 1.5 hours
**Dependencies:** Phase 3 (Kiosk) + Phase 4 (Stripe) complete
**Tech stack:** Next.js, Supabase, Supabase Realtime, Tailwind CSS

## Dashboard Sections

### 1. Today's Bookings (Primary View)
**Purpose:** Monitor and manage all bookings for the current day

### 2. Cash Payment Confirmation
**Purpose:** Mark cash payments as received (unblocks kiosk waiting screen)

### 3. Member Management
**Purpose:** View, edit, and create member profiles

### 4. Payment History
**Purpose:** Financial reporting and reconciliation

## Authentication (15 mins)

### Supabase Auth Setup

```bash
# Enable Supabase Auth in project dashboard
# Create admin user via SQL:

INSERT INTO admin_users (email, role, is_active)
VALUES ('admin@honourandglory.co.uk', 'owner', true);
```

### Login Page: `/admin/login`

```typescript
// apps/web/src/app/admin/login/page.tsx

'use client';

import { useState } from 'react';
import { createClient } from '@/lib/supabase-client';
import { useRouter } from 'next/navigation';

export default function AdminLogin() {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [error, setError] = useState('');
  const [loading, setLoading] = useState(false);
  const router = useRouter();
  const supabase = createClient();

  const handleLogin = async (e: React.FormEvent) => {
    e.preventDefault();
    setError('');
    setLoading(true);

    const { error: authError } = await supabase.auth.signInWithPassword({
      email,
      password,
    });

    if (authError) {
      setError(authError.message);
      setLoading(false);
      return;
    }

    router.push('/admin');
  };

  return (
    <div className="min-h-screen flex items-center justify-center bg-gray-50">
      <div className="max-w-md w-full space-y-8 p-8 bg-white rounded-lg shadow">
        <div>
          <h2 className="text-3xl font-bold text-center">H&G Admin</h2>
          <p className="text-center text-gray-600 mt-2">Staff Dashboard</p>
        </div>
        
        <form onSubmit={handleLogin} className="space-y-6">
          {error && (
            <div className="bg-red-50 text-red-600 p-3 rounded">
              {error}
            </div>
          )}
          
          <div>
            <label className="block text-sm font-medium text-gray-700">
              Email
            </label>
            <input
              type="email"
              value={email}
              onChange={(e) => setEmail(e.target.value)}
              className="mt-1 block w-full rounded border-gray-300 shadow-sm"
              required
            />
          </div>
          
          <div>
            <label className="block text-sm font-medium text-gray-700">
              Password
            </label>
            <input
              type="password"
              value={password}
              onChange={(e) => setPassword(e.target.value)}
              className="mt-1 block w-full rounded border-gray-300 shadow-sm"
              required
            />
          </div>
          
          <button
            type="submit"
            disabled={loading}
            className="w-full bg-black text-white py-3 rounded hover:bg-gray-800"
          >
            {loading ? 'Logging in...' : 'Log In'}
          </button>
        </form>
      </div>
    </div>
  );
}
```

### Protected Route Wrapper

```typescript
// apps/web/src/components/admin/ProtectedRoute.tsx

'use client';

import { useEffect, useState } from 'react';
import { useRouter } from 'next/navigation';
import { createClient } from '@/lib/supabase-client';

export default function ProtectedRoute({ children }: { children: React.ReactNode }) {
  const [loading, setLoading] = useState(true);
  const [authenticated, setAuthenticated] = useState(false);
  const router = useRouter();
  const supabase = createClient();

  useEffect(() => {
    checkAuth();
  }, []);

  const checkAuth = async () => {
    const { data: { session } } = await supabase.auth.getSession();
    
    if (!session) {
      router.push('/admin/login');
      return;
    }

    // Check if user is in admin_users table
    const { data: adminUser } = await supabase
      .from('admin_users')
      .select('*')
      .eq('email', session.user.email)
      .eq('is_active', true)
      .single();

    if (!adminUser) {
      await supabase.auth.signOut();
      router.push('/admin/login');
      return;
    }

    setAuthenticated(true);
    setLoading(false);
  };

  if (loading) {
    return <div className="min-h-screen flex items-center justify-center">Loading...</div>;
  }

  if (!authenticated) {
    return null;
  }

  return <>{children}</>;
}
```

## Today's Bookings Dashboard (40 mins)

### Main Dashboard: `/admin`

```typescript
// apps/web/src/app/admin/page.tsx

'use client';

import { useEffect, useState } from 'react';
import { createClient } from '@/lib/supabase-client';
import ProtectedRoute from '@/components/admin/ProtectedRoute';

type Booking = {
  id: string;
  member: {
    first_name: string;
    last_name: string;
    phone: string;
  };
  class: {
    name: string;
  };
  session_start_time: string;
  payment_status: 'pending' | 'paid' | 'refunded';
  status: 'confirmed' | 'cancelled' | 'attended' | 'no-show';
  amount_gbp: number;
  created_via: 'kiosk' | 'admin' | 'online';
};

export default function AdminDashboard() {
  const [bookings, setBookings] = useState<Booking[]>([]);
  const [loading, setLoading] = useState(true);
  const supabase = createClient();

  useEffect(() => {
    loadTodaysBookings();
    
    // Subscribe to realtime updates
    const channel = supabase
      .channel('bookings-changes')
      .on(
        'postgres_changes',
        {
          event: '*',
          schema: 'public',
          table: 'bookings',
          filter: `session_date=eq.${new Date().toISOString().split('T')[0]}`
        },
        () => {
          loadTodaysBookings();
        }
      )
      .subscribe();

    return () => {
      supabase.removeChannel(channel);
    };
  }, []);

  const loadTodaysBookings = async () => {
    const today = new Date().toISOString().split('T')[0];
    
    const { data, error } = await supabase
      .from('bookings')
      .select(`
        *,
        member:members(first_name, last_name, phone),
        class:classes(name)
      `)
      .eq('session_date', today)
      .order('session_start_time', { ascending: true });

    if (error) {
      console.error('Failed to load bookings:', error);
      return;
    }

    setBookings(data || []);
    setLoading(false);
  };

  const confirmCashPayment = async (bookingId: string) => {
    const { data: { user } } = await supabase.auth.getUser();
    
    const response = await fetch('/api/payments/confirm-cash', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ bookingId, staffUserId: user?.id }),
    });

    if (response.ok) {
      loadTodaysBookings();
    }
  };

  const markAttended = async (bookingId: string) => {
    await supabase
      .from('bookings')
      .update({ 
        status: 'attended',
        checked_in_at: new Date().toISOString()
      })
      .eq('id', bookingId);
    
    loadTodaysBookings();
  };

  if (loading) {
    return (
      <ProtectedRoute>
        <div className="p-8">Loading...</div>
      </ProtectedRoute>
    );
  }

  const pendingPayments = bookings.filter(b => b.payment_status === 'pending');
  const confirmedBookings = bookings.filter(b => b.payment_status === 'paid');

  return (
    <ProtectedRoute>
      <div className="min-h-screen bg-gray-50">
        <header className="bg-white shadow">
          <div className="max-w-7xl mx-auto px-4 py-6 flex justify-between items-center">
            <h1 className="text-3xl font-bold">Today's Bookings</h1>
            <div className="text-sm text-gray-600">
              {new Date().toLocaleDateString('en-GB', {
                weekday: 'long',
                day: 'numeric',
                month: 'long',
                year: 'numeric'
              })}
            </div>
          </div>
        </header>

        <main className="max-w-7xl mx-auto px-4 py-8">
          {/* Summary Cards */}
          <div className="grid grid-cols-3 gap-6 mb-8">
            <div className="bg-white p-6 rounded-lg shadow">
              <div className="text-2xl font-bold">{bookings.length}</div>
              <div className="text-gray-600">Total Bookings</div>
            </div>
            <div className="bg-yellow-50 p-6 rounded-lg shadow">
              <div className="text-2xl font-bold text-yellow-600">{pendingPayments.length}</div>
              <div className="text-gray-600">Awaiting Payment</div>
            </div>
            <div className="bg-green-50 p-6 rounded-lg shadow">
              <div className="text-2xl font-bold text-green-600">{confirmedBookings.length}</div>
              <div className="text-gray-600">Confirmed</div>
            </div>
          </div>

          {/* Pending Cash Payments */}
          {pendingPayments.length > 0 && (
            <div className="mb-8">
              <h2 className="text-xl font-bold mb-4">⚠️ Awaiting Cash Payment</h2>
              <div className="bg-white rounded-lg shadow">
                {pendingPayments.map((booking) => (
                  <div
                    key={booking.id}
                    className="border-b last:border-b-0 p-4 flex justify-between items-center"
                  >
                    <div>
                      <div className="font-semibold">
                        {booking.member.first_name} {booking.member.last_name}
                      </div>
                      <div className="text-sm text-gray-600">
                        {booking.class.name} • {booking.session_start_time}
                      </div>
                    </div>
                    <div className="flex items-center gap-4">
                      <div className="font-bold">£{booking.amount_gbp.toFixed(2)}</div>
                      <button
                        onClick={() => confirmCashPayment(booking.id)}
                        className="bg-green-600 text-white px-4 py-2 rounded hover:bg-green-700"
                      >
                        Confirm Cash Received
                      </button>
                    </div>
                  </div>
                ))}
              </div>
            </div>
          )}

          {/* All Bookings List */}
          <div>
            <h2 className="text-xl font-bold mb-4">All Bookings Today</h2>
            <div className="bg-white rounded-lg shadow overflow-hidden">
              <table className="min-w-full divide-y divide-gray-200">
                <thead className="bg-gray-50">
                  <tr>
                    <th className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">
                      Time
                    </th>
                    <th className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">
                      Member
                    </th>
                    <th className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">
                      Class
                    </th>
                    <th className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">
                      Payment
                    </th>
                    <th className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">
                      Status
                    </th>
                    <th className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">
                      Actions
                    </th>
                  </tr>
                </thead>
                <tbody className="bg-white divide-y divide-gray-200">
                  {bookings.map((booking) => (
                    <tr key={booking.id}>
                      <td className="px-6 py-4 whitespace-nowrap text-sm">
                        {booking.session_start_time}
                      </td>
                      <td className="px-6 py-4 whitespace-nowrap">
                        <div className="text-sm font-medium">
                          {booking.member.first_name} {booking.member.last_name}
                        </div>
                        <div className="text-sm text-gray-500">{booking.member.phone}</div>
                      </td>
                      <td className="px-6 py-4 whitespace-nowrap text-sm">
                        {booking.class.name}
                      </td>
                      <td className="px-6 py-4 whitespace-nowrap">
                        <span
                          className={`px-2 py-1 text-xs rounded ${
                            booking.payment_status === 'paid'
                              ? 'bg-green-100 text-green-800'
                              : 'bg-yellow-100 text-yellow-800'
                          }`}
                        >
                          {booking.payment_status}
                        </span>
                      </td>
                      <td className="px-6 py-4 whitespace-nowrap text-sm">
                        {booking.status}
                      </td>
                      <td className="px-6 py-4 whitespace-nowrap text-sm">
                        {booking.status === 'confirmed' && (
                          <button
                            onClick={() => markAttended(booking.id)}
                            className="text-blue-600 hover:text-blue-800"
                          >
                            Mark Attended
                          </button>
                        )}
                      </td>
                    </tr>
                  ))}
                </tbody>
              </table>
            </div>
          </div>
        </main>
      </div>
    </ProtectedRoute>
  );
}
```

## Member Management (30 mins)

### Members List Page: `/admin/members`

```typescript
// apps/web/src/app/admin/members/page.tsx

'use client';

import { useState, useEffect } from 'react';
import { createClient } from '@/lib/supabase-client';
import ProtectedRoute from '@/components/admin/ProtectedRoute';
import Link from 'next/link';

export default function MembersPage() {
  const [members, setMembers] = useState([]);
  const [search, setSearch] = useState('');
  const [loading, setLoading] = useState(true);
  const supabase = createClient();

  useEffect(() => {
    loadMembers();
  }, [search]);

  const loadMembers = async () => {
    let query = supabase
      .from('members')
      .select('*')
      .order('created_at', { ascending: false });

    if (search) {
      query = query.or(
        `first_name.ilike.%${search}%,last_name.ilike.%${search}%,phone.ilike.%${search}%`
      );
    }

    const { data, error } = await query;

    if (!error) {
      setMembers(data || []);
    }
    setLoading(false);
  };

  return (
    <ProtectedRoute>
      <div className="min-h-screen bg-gray-50">
        <header className="bg-white shadow">
          <div className="max-w-7xl mx-auto px-4 py-6">
            <h1 className="text-3xl font-bold">Members</h1>
          </div>
        </header>

        <main className="max-w-7xl mx-auto px-4 py-8">
          <div className="mb-6 flex gap-4">
            <input
              type="text"
              placeholder="Search members..."
              value={search}
              onChange={(e) => setSearch(e.target.value)}
              className="flex-1 rounded border-gray-300 shadow-sm"
            />
            <Link
              href="/admin/members/new"
              className="bg-black text-white px-6 py-2 rounded hover:bg-gray-800"
            >
              Add Member
            </Link>
          </div>

          <div className="bg-white rounded-lg shadow overflow-hidden">
            <table className="min-w-full divide-y divide-gray-200">
              <thead className="bg-gray-50">
                <tr>
                  <th className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">
                    Name
                  </th>
                  <th className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">
                    Contact
                  </th>
                  <th className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">
                    Age Group
                  </th>
                  <th className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">
                    Status
                  </th>
                  <th className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">
                    Actions
                  </th>
                </tr>
              </thead>
              <tbody className="divide-y divide-gray-200">
                {members.map((member: any) => (
                  <tr key={member.id}>
                    <td className="px-6 py-4">
                      {member.first_name} {member.last_name}
                    </td>
                    <td className="px-6 py-4">
                      <div className="text-sm">{member.email}</div>
                      <div className="text-sm text-gray-500">{member.phone}</div>
                    </td>
                    <td className="px-6 py-4 capitalize">{member.age_group || '-'}</td>
                    <td className="px-6 py-4">
                      <span
                        className={`px-2 py-1 text-xs rounded ${
                          member.status === 'active'
                            ? 'bg-green-100 text-green-800'
                            : 'bg-gray-100 text-gray-800'
                        }`}
                      >
                        {member.status}
                      </span>
                    </td>
                    <td className="px-6 py-4">
                      <Link
                        href={`/admin/members/${member.id}`}
                        className="text-blue-600 hover:text-blue-800"
                      >
                        View
                      </Link>
                    </td>
                  </tr>
                ))}
              </tbody>
            </table>
          </div>
        </main>
      </div>
    </ProtectedRoute>
  );
}
```

## Payment History (15 mins)

### Payments Page: `/admin/payments`

Basic implementation for financial reconciliation:

```typescript
// apps/web/src/app/admin/payments/page.tsx

'use client';

import { useState, useEffect } from 'react';
import { createClient } from '@/lib/supabase-client';
import ProtectedRoute from '@/components/admin/ProtectedRoute';

export default function PaymentsPage() {
  const [payments, setPayments] = useState([]);
  const [dateRange, setDateRange] = useState('today');
  const supabase = createClient();

  useEffect(() => {
    loadPayments();
  }, [dateRange]);

  const loadPayments = async () => {
    const { data } = await supabase
      .from('payments')
      .select(`
        *,
        member:members(first_name, last_name),
        booking:bookings(class:classes(name))
      `)
      .order('created_at', { ascending: false })
      .limit(100);

    setPayments(data || []);
  };

  const totalRevenue = payments.reduce(
    (sum, p: any) => sum + (p.status === 'completed' ? p.amount_gbp : 0),
    0
  );

  return (
    <ProtectedRoute>
      <div className="min-h-screen bg-gray-50">
        <header className="bg-white shadow">
          <div className="max-w-7xl mx-auto px-4 py-6">
            <h1 className="text-3xl font-bold">Payment History</h1>
          </div>
        </header>

        <main className="max-w-7xl mx-auto px-4 py-8">
          <div className="bg-white p-6 rounded-lg shadow mb-6">
            <div className="text-2xl font-bold">£{totalRevenue.toFixed(2)}</div>
            <div className="text-gray-600">Total Revenue (Last 100 payments)</div>
          </div>

          <div className="bg-white rounded-lg shadow overflow-hidden">
            <table className="min-w-full divide-y divide-gray-200">
              <thead className="bg-gray-50">
                <tr>
                  <th className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">
                    Date
                  </th>
                  <th className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">
                    Member
                  </th>
                  <th className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">
                    Class
                  </th>
                  <th className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">
                    Method
                  </th>
                  <th className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">
                    Amount
                  </th>
                  <th className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">
                    Status
                  </th>
                </tr>
              </thead>
              <tbody className="divide-y divide-gray-200">
                {payments.map((payment: any) => (
                  <tr key={payment.id}>
                    <td className="px-6 py-4 text-sm">
                      {new Date(payment.created_at).toLocaleDateString()}
                    </td>
                    <td className="px-6 py-4">
                      {payment.member?.first_name} {payment.member?.last_name}
                    </td>
                    <td className="px-6 py-4 text-sm">
                      {payment.booking?.class?.name || '-'}
                    </td>
                    <td className="px-6 py-4 capitalize text-sm">
                      {payment.payment_method}
                    </td>
                    <td className="px-6 py-4 font-medium">
                      £{payment.amount_gbp.toFixed(2)}
                    </td>
                    <td className="px-6 py-4">
                      <span
                        className={`px-2 py-1 text-xs rounded ${
                          payment.status === 'completed'
                            ? 'bg-green-100 text-green-800'
                            : payment.status === 'refunded'
                            ? 'bg-red-100 text-red-800'
                            : 'bg-yellow-100 text-yellow-800'
                        }`}
                      >
                        {payment.status}
                      </span>
                    </td>
                  </tr>
                ))}
              </tbody>
            </table>
          </div>
        </main>
      </div>
    </ProtectedRoute>
  );
}
```

## Navigation & Layout (10 mins)

### Admin Layout Component

```typescript
// apps/web/src/components/admin/AdminLayout.tsx

import Link from 'next/link';
import { usePathname } from 'next/navigation';

export default function AdminLayout({ children }: { children: React.ReactNode }) {
  const pathname = usePathname();
  
  const navItems = [
    { href: '/admin', label: 'Dashboard' },
    { href: '/admin/members', label: 'Members' },
    { href: '/admin/payments', label: 'Payments' },
  ];

  return (
    <div className="flex">
      <nav className="w-64 min-h-screen bg-gray-900 text-white">
        <div className="p-6">
          <h1 className="text-xl font-bold">H&G Admin</h1>
        </div>
        <ul>
          {navItems.map((item) => (
            <li key={item.href}>
              <Link
                href={item.href}
                className={`block px-6 py-3 hover:bg-gray-800 ${
                  pathname === item.href ? 'bg-gray-800' : ''
                }`}
              >
                {item.label}
              </Link>
            </li>
          ))}
        </ul>
      </nav>
      <div className="flex-1">{children}</div>
    </div>
  );
}
```

## Phase 5 Implementation Checklist

### Authentication (15 mins)
- [ ] Enable Supabase Auth in project
- [ ] Create admin user in `admin_users` table
- [ ] Build login page (`/admin/login`)
- [ ] Build ProtectedRoute wrapper component
- [ ] Test login flow

### Today's Bookings Dashboard (40 mins)
- [ ] Build main dashboard page (`/admin`)
- [ ] Implement real-time booking subscription
- [ ] Add summary cards (total, pending, confirmed)
- [ ] Build pending cash payments section
- [ ] Build bookings table with all statuses
- [ ] Add "Confirm Cash Payment" button
- [ ] Add "Mark Attended" functionality
- [ ] Test real-time updates when kiosk creates booking

### Member Management (30 mins)
- [ ] Build members list page (`/admin/members`)
- [ ] Implement search functionality
- [ ] Build member detail page (`/admin/members/[id]`)
- [ ] Build new member form (reuse from kiosk)
- [ ] Test CRUD operations

### Payment History (15 mins)
- [ ] Build payments page (`/admin/payments`)
- [ ] Display total revenue card
- [ ] Build payments table with filters
- [ ] Add date range selector
- [ ] Test revenue calculations

### Navigation & Layout (10 mins)
- [ ] Build AdminLayout component
- [ ] Add navigation sidebar
- [ ] Apply to all admin pages
- [ ] Add logout button

### Testing (10 mins)
- [ ] End-to-end: kiosk booking → admin confirms cash
- [ ] Real-time updates when new booking created
- [ ] Member search and profile editing
- [ ] Payment history accuracy
- [ ] Authentication logout/session expiry

## Security Notes

- Admin routes protected by Supabase Auth + `admin_users` table check
- RLS policies prevent kiosk from accessing admin data
- Service role key only used server-side
- Staff passwords should be strong (enforce policy in Supabase Auth)
- Consider 2FA for owner role (Supabase Auth supports it)

## Deployment

- Deploy to Vercel (same project as kiosk)
- Access via `/admin` route
- Desktop/laptop recommended for admin dashboard
- Mobile responsive but optimized for larger screens

## Future Enhancements (Post-MVP)

- Class schedule management (edit times, cancel classes)
- Bulk member import (CSV upload)
- Email/SMS notifications (booking confirmations)
- Revenue reports (daily/weekly/monthly)
- Capacity management (waiting lists)
- Multi-location support (if H&G expands)

## Total Implementation Time

- Authentication: 15 mins
- Today's Bookings: 40 mins
- Member Management: 30 mins
- Payment History: 15 mins
- Navigation: 10 mins
- Testing: 10 mins

**Total:** ~2 hours (slightly over 1.5h estimate, includes buffer for testing)

---

**All 5 phases now fully documented. Total remaining execution time: ~6 hours when deployment succeeds.**
