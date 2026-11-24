# Booking Management System - Project Plan

## Overview
A React-based booking management boilerplate with role-based access control, audit trails, and Philippines-specific payment handling (external).

---

## 1. Tech Stack Recommendations

### Frontend
| Technology | Version | Purpose |
|------------|---------|---------|
| React | 18+ | Core UI framework |
| TypeScript | 5+ | Type safety |
| Vite | 5+ | Build tool & dev server |
| React Router | 6+ | Client-side routing |
| TanStack Query (React Query) | 5+ | Server state management |
| Zustand / Redux Toolkit | Latest | Client state management |
| Tailwind CSS | 3+ | Styling |
| shadcn/ui / Radix UI | Latest | Accessible component primitives |
| React Hook Form + Zod | Latest | Form handling & validation |
| Axios | Latest | HTTP client |
| date-fns / dayjs | Latest | Date manipulation |
| Chart.js / Recharts | Latest | Analytics dashboards |

### Backend Options (Choose One)
| Option | Pros | Cons | Recommended For |
|--------|------|------|-----------------|
| **Supabase** | PostgreSQL, Auth, Realtime, Storage, Edge Functions | Vendor lock-in | Quick start, real-time needs |
| **Firebase** | Auth, Firestore, Functions, Hosting | NoSQL, pricing at scale | Mobile-first, offline sync |
| **Appwrite** | Self-hostable, Auth, Database, Functions, Storage | Smaller community | Full control, self-hosting |
| **Node.js + Express/NestJS + PostgreSQL** | Full control, standard SQL | More setup | Custom requirements, team expertise |
| **Hasura + PostgreSQL** | Instant GraphQL, Auth, Permissions | Learning curve | GraphQL preference |

### Database (PostgreSQL Recommended)
- **Primary**: PostgreSQL (via Supabase, Neon, Railway, or self-hosted)
- **Why**: Relational data, complex queries, audit trails, mature ecosystem
- **Hosting**: Supabase (free tier), Neon (serverless), Railway, Render, or AWS RDS

### Authentication
- **Supabase Auth** / **Clerk** / **Auth0** / **Custom JWT**
- **Recommendation**: Supabase Auth (integrated with DB, supports Row Level Security)

### Deployment
- **Frontend**: Vercel, Netlify, or Cloudflare Pages
- **Backend**: Same as DB hosting or separate (Railway, Render, Fly.io)
- **CI/CD**: GitHub Actions

---

## 2. Project Structure

```
booking-management/
├── .github/
│   └── workflows/           # CI/CD pipelines
├── public/                  # Static assets
├── src/
│   ├── app/                 # App-level configuration
│   │   ├── providers/       # Context providers (Auth, Query, Theme)
│   │   ├── routes/          # Route definitions & guards
│   │   └── layout/          # Root layout components
│   ├── features/            # Feature-based modules
│   │   ├── auth/            # Authentication (login, register, password reset)
│   │   ├── bookings/        # Booking management
│   │   ├── users/           # User management (admin only)
│   │   ├── payments/        # Payment recording (manual entry)
│   │   ├── audit/           # Audit logs / history
│   │   ├── dashboard/       # Role-based dashboards
│   │   └── settings/        # System settings
│   ├── shared/              # Shared code across features
│   │   ├── components/      # Reusable UI components
│   │   ├── hooks/           # Custom hooks
│   │   ├── utils/           # Utility functions
│   │   ├── types/           # TypeScript types
│   │   ├── constants/       # App constants
│   │   ├── api/             # API client & endpoints
│   │   └── validators/      # Zod schemas
│   ├── styles/              # Global styles, Tailwind config
│   └── main.tsx             # Entry point
├── supabase/                # Supabase config (if using Supabase)
│   ├── migrations/          # Database migrations
│   ├── functions/           # Edge functions
│   └── seed.sql             # Seed data
├── tests/                   # E2E & integration tests
├── .env.example             # Environment variables template
├── package.json
├── tsconfig.json
├── vite.config.ts
├── tailwind.config.js
└── README.md
```

---

## 3. Database Schema (PostgreSQL)

### Core Tables

```sql
-- Users & Authentication
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    full_name VARCHAR(255),
    phone VARCHAR(50),
    avatar_url TEXT,
    role user_role NOT NULL DEFAULT 'user',
    status user_status NOT NULL DEFAULT 'active',
    email_verified_at TIMESTAMPTZ,
    last_login_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    deleted_at TIMESTAMPTZ
);

CREATE TYPE user_role AS ENUM ('super_admin', 'admin', 'user');
CREATE TYPE user_status AS ENUM ('active', 'inactive', 'suspended', 'pending');

-- Booking Resources (what can be booked)
CREATE TABLE resources (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    description TEXT,
    type resource_type NOT NULL,
    capacity INTEGER DEFAULT 1,
    location VARCHAR(500),
    amenities JSONB DEFAULT '[]',
    pricing JSONB, -- { "hourly": 500, "daily": 3000, "currency": "PHP" }
    status resource_status NOT NULL DEFAULT 'available',
    created_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TYPE resource_type AS ENUM ('room', 'vehicle', 'equipment', 'space', 'service');
CREATE TYPE resource_status AS ENUM ('available', 'maintenance', 'unavailable', 'archived');

-- Bookings
CREATE TABLE bookings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    booking_number VARCHAR(50) UNIQUE NOT NULL, -- e.g., BK-2024-001234
    resource_id UUID NOT NULL REFERENCES resources(id),
    user_id UUID NOT NULL REFERENCES users(id), -- who made the booking
    booker_id UUID REFERENCES users(id), -- who it's for (if different)
    status booking_status NOT NULL DEFAULT 'pending',
    start_time TIMESTAMPTZ NOT NULL,
    end_time TIMESTAMPTZ NOT NULL,
    timezone VARCHAR(50) DEFAULT 'Asia/Manila',
    total_amount DECIMAL(12,2) DEFAULT 0,
    currency VARCHAR(3) DEFAULT 'PHP',
    notes TEXT,
    cancellation_reason TEXT,
    cancelled_at TIMESTAMPTZ,
    cancelled_by UUID REFERENCES users(id),
    confirmed_at TIMESTAMPTZ,
    confirmed_by UUID REFERENCES users(id),
    checked_in_at TIMESTAMPTZ,
    checked_out_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TYPE booking_status AS ENUM (
    'pending', 'confirmed', 'checked_in', 'checked_out', 
    'completed', 'cancelled', 'no_show', 'disputed'
);

-- Payments (Manual recording - external payment processing)
CREATE TABLE payments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    payment_number VARCHAR(50) UNIQUE NOT NULL, -- e.g., PAY-2024-001234
    booking_id UUID NOT NULL REFERENCES bookings(id),
    user_id UUID NOT NULL REFERENCES users(id), -- who recorded the payment
    amount DECIMAL(12,2) NOT NULL,
    currency VARCHAR(3) DEFAULT 'PHP',
    method payment_method NOT NULL,
    reference_number VARCHAR(255), -- GCash ref, bank transfer ref, etc.
    status payment_status NOT NULL DEFAULT 'pending',
    paid_at TIMESTAMPTZ NOT NULL,
    notes TEXT,
    receipt_url TEXT, -- uploaded receipt image
    verified_at TIMESTAMPTZ,
    verified_by UUID REFERENCES users(id),
    refunded_at TIMESTAMPTZ,
    refunded_amount DECIMAL(12,2),
    refund_reason TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TYPE payment_method AS ENUM (
    'cash', 'gcash', 'paymaya', 'bank_transfer', 'credit_card', 
    'debit_card', 'check', 'other'
);
CREATE TYPE payment_status AS ENUM (
    'pending', 'verified', 'partial', 'refunded', 'failed', 'disputed'
);

-- Audit Logs (Comprehensive history)
CREATE TABLE audit_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    entity_type VARCHAR(100) NOT NULL, -- 'booking', 'user', 'resource', 'payment', 'setting'
    entity_id UUID NOT NULL,
    action audit_action NOT NULL,
    actor_id UUID REFERENCES users(id), -- who performed the action
    actor_role user_role, -- role at time of action
    old_values JSONB, -- previous state
    new_values JSONB, -- new state
    changed_fields TEXT[], -- array of field names that changed
    ip_address INET,
    user_agent TEXT,
    metadata JSONB DEFAULT '{}', -- additional context
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TYPE audit_action AS ENUM (
    'create', 'update', 'delete', 'restore', 
    'status_change', 'assign', 'unassign', 
    'login', 'logout', 'password_change', 'role_change',
    'payment_record', 'payment_verify', 'payment_refund',
    'booking_confirm', 'booking_cancel', 'booking_checkin', 'booking_checkout',
    'resource_create', 'resource_update', 'resource_archive',
    'user_create', 'user_update', 'user_suspend', 'user_activate',
    'settings_change', 'bulk_action', 'export', 'import'
);

-- Indexes for performance
CREATE INDEX idx_bookings_user_id ON bookings(user_id);
CREATE INDEX idx_bookings_resource_id ON bookings(resource_id);
CREATE INDEX idx_bookings_status ON bookings(status);
CREATE INDEX idx_bookings_date_range ON bookings(start_time, end_time);
CREATE INDEX idx_payments_booking_id ON payments(booking_id);
CREATE INDEX idx_payments_user_id ON payments(user_id);
CREATE INDEX idx_audit_logs_entity ON audit_logs(entity_type, entity_id);
CREATE INDEX idx_audit_logs_actor ON audit_logs(actor_id);
CREATE INDEX idx_audit_logs_created_at ON audit_logs(created_at);
CREATE INDEX idx_audit_logs_action ON audit_logs(action);

-- Row Level Security (RLS) Policies - Supabase
ALTER TABLE users ENABLE ROW LEVEL SECURITY;
ALTER TABLE resources ENABLE ROW LEVEL SECURITY;
ALTER TABLE bookings ENABLE ROW LEVEL SECURITY;
ALTER TABLE payments ENABLE ROW LEVEL SECURITY;
ALTER TABLE audit_logs ENABLE ROW LEVEL SECURITY;

-- Example RLS Policies
-- Users can read their own profile, admins can read all
CREATE POLICY "Users can view own profile" ON users
    FOR SELECT USING (auth.uid() = id OR 
        EXISTS (SELECT 1 FROM users WHERE id = auth.uid() AND role IN ('admin', 'super_admin')));

-- Users can view their own bookings, admins can view all
CREATE POLICY "Users can view own bookings" ON bookings
    FOR SELECT USING (user_id = auth.uid() OR 
        EXISTS (SELECT 1 FROM users WHERE id = auth.uid() AND role IN ('admin', 'super_admin')));

-- Only admins/super_admins can manage users
CREATE POLICY "Admins can manage users" ON users
    FOR ALL USING (EXISTS (SELECT 1 FROM users WHERE id = auth.uid() AND role IN ('admin', 'super_admin')));

-- Audit logs: only super_admin can view all, admin can view non-security
CREATE POLICY "Super admin can view all audit logs" ON audit_logs
    FOR SELECT USING (EXISTS (SELECT 1 FROM users WHERE id = auth.uid() AND role = 'super_admin'));
```

---

## 4. Role-Based Access Control (RBAC)

### Permission Matrix

| Feature | Super Admin | Admin | User |
|---------|-------------|-------|------|
| **Dashboard** | Full analytics | Full analytics | Own bookings only |
| **Bookings** | CRUD all | CRUD all | Create own, view own, cancel own |
| **Resources** | CRUD all | CRUD all | View available only |
| **Users** | CRUD all | CRUD (except super_admin) | View own profile only |
| **Payments** | Record, verify, refund all | Record, verify all | Record own, view own |
| **Audit Logs** | View all | View (exclude security) | View own actions only |
| **Settings** | Full access | Limited (no security) | None |
| **Security** | Full (roles, RLS, keys) | None | None |

### Implementation Approach
```typescript
// shared/types/rbac.ts
export type Role = 'super_admin' | 'admin' | 'user';
export type Permission = 
  | 'bookings:create' | 'bookings:read' | 'bookings:update' | 'bookings:delete'
  | 'bookings:confirm' | 'bookings:cancel' | 'bookings:checkin' | 'bookings:checkout'
  | 'resources:create' | 'resources:read' | 'resources:update' | 'resources:delete'
  | 'users:create' | 'users:read' | 'users:update' | 'users:delete' | 'users:manage_roles'
  | 'payments:create' | 'payments:read' | 'payments:verify' | 'payments:refund'
  | 'audit:read' | 'audit:read_security' | 'audit:export'
  | 'settings:read' | 'settings:update' | 'settings:security';

const rolePermissions: Record<Role, Permission[]> = {
  super_admin: ['*'], // All permissions
  admin: [
    'bookings:*', 'resources:*', 'users:read', 'users:create', 'users:update',
    'payments:*', 'audit:read', 'audit:export', 'settings:read', 'settings:update'
  ],
  user: [
    'bookings:create', 'bookings:read:own', 'bookings:cancel:own',
    'resources:read:available', 'payments:create:own', 'payments:read:own',
    'audit:read:own'
  ]
};
```

---

## 5. API Endpoints Design

### Authentication
```
POST   /api/auth/login              # Login
POST   /api/auth/register           # Register (public or invite-only)
POST   /api/auth/refresh            # Refresh access token
POST   /api/auth/logout             # Logout
POST   /api/auth/forgot-password    # Request password reset
POST   /api/auth/reset-password     # Reset password
GET    /api/auth/me                 # Get current user profile
PUT    /api/auth/me                 # Update profile
```

### Bookings
```
GET    /api/bookings                # List bookings (with filters, pagination)
POST   /api/bookings                # Create booking
GET    /api/bookings/:id            # Get booking details
PUT    /api/bookings/:id            # Update booking
DELETE /api/bookings/:id            # Delete booking (soft)
POST   /api/bookings/:id/confirm    # Confirm booking
POST   /api/bookings/:id/cancel     # Cancel booking
POST   /api/bookings/:id/checkin    # Check-in
POST   /api/bookings/:id/checkout   # Check-out
GET    /api/bookings/:id/history    # Booking audit history
GET    /api/bookings/availability   # Check resource availability
```

### Resources
```
GET    /api/resources               # List resources
POST   /api/resources               # Create resource (admin+)
GET    /api/resources/:id           # Get resource details
PUT    /api/resources/:id           # Update resource (admin+)
DELETE /api/resources/:id           # Archive resource (admin+)
GET    /api/resources/:id/bookings  # Resource bookings
```

### Users (Admin+)
```
GET    /api/users                   # List users
POST   /api/users                   # Create user (super_admin only for admin roles)
GET    /api/users/:id               # Get user details
PUT    /api/users/:id               # Update user
DELETE /api/users/:id               # Deactivate user
PUT    /api/users/:id/role          # Change role (super_admin only)
PUT    /api/users/:id/status        # Activate/suspend user
```

### Payments
```
GET    /api/payments                # List payments
POST   /api/payments                # Record payment
GET    /api/payments/:id            # Get payment details
PUT    /api/payments/:id            # Update payment
POST   /api/payments/:id/verify     # Verify payment (admin+)
POST   /api/payments/:id/refund     # Refund payment (admin+)
GET    /api/bookings/:id/payments   # Payments for a booking
```

### Audit Logs
```
GET    /api/audit-logs              # List audit logs (with filters)
GET    /api/audit-logs/:id          # Get audit log details
GET    /api/audit-logs/export       # Export audit logs (CSV/PDF)
GET    /api/entities/:type/:id/history  # Entity-specific history
```

### Dashboard/Analytics
```
GET    /api/dashboard/stats         # Role-based dashboard stats
GET    /api/dashboard/bookings-trend # Booking trends
GET    /api/dashboard/revenue       # Revenue analytics
GET    /api/dashboard/resource-utilization # Resource utilization
```

---

## 6. Frontend Architecture

### State Management Strategy
- **Server State**: TanStack Query (caching, synchronization, background updates)
- **Client State**: Zustand (auth state, UI state, filters, form state)
- **Form State**: React Hook Form + Zod validation

### Routing Structure
```
/                                    # Redirect to dashboard
/login                               # Login page
/register                            # Registration (if public)
/forgot-password                     # Password reset request
/reset-password/:token               # Password reset form

/dashboard                           # Role-based dashboard
/bookings                            # Booking list
/bookings/new                        # Create booking
/bookings/:id                        # Booking details
/bookings/:id/edit                   # Edit booking

/resources                           # Resource list (admin+)
/resources/new                       # Create resource (admin+)
/resources/:id                       # Resource details
/resources/:id/edit                  # Edit resource (admin+)

/users                               # User management (admin+)
/users/:id                           # User details
/users/:id/edit                      # Edit user (admin+)

/payments                            # Payment list
/payments/new                        # Record payment
/payments/:id                        # Payment details

/audit-logs                          # Audit logs (admin+)
/audit-logs/:id                      # Audit log details

/settings                            # Settings (admin+)
/settings/general                    # General settings
/settings/security                   # Security settings (super_admin only)
/settings/notifications              # Notification settings
/profile                             # User profile
```

### Component Architecture
```
features/bookings/
├── components/
│   ├── BookingCard.tsx
│   ├── BookingForm.tsx
│   ├── BookingList.tsx
│   ├── BookingDetails.tsx
│   ├── BookingCalendar.tsx
│   ├── BookingFilters.tsx
│   ├── BookingStatusBadge.tsx
│   └── AvailabilityChecker.tsx
├── hooks/
│   ├── useBookings.ts
│   ├── useBooking.ts
│   ├── useCreateBooking.ts
│   └── useBookingActions.ts
├── api/
│   └── bookings.ts
├── types/
│   └── booking.ts
└── validators/
    └── booking.ts
```

---

## 7. Key Features Implementation Details

### 7.1 Booking Flow
1. **User** views available resources (calendar/list view)
2. **User** selects resource, date/time, fills booking form
3. **System** checks availability (prevent double booking)
4. **User** submits → **Pending** status
5. **Admin** reviews → **Confirmed** or **Rejected**
6. **User** receives notification (email/push)
7. **Check-in/Check-out** on booking dates
8. **Completion** → **Completed** status
9. **Payment** recorded manually (cash, GCash, bank transfer, etc.)

### 7.2 Payment Recording (Philippines Context)
```typescript
// Payment methods specific to Philippines
const PH_PAYMENT_METHODS = [
  { value: 'cash', label: 'Cash', icon: '💵' },
  { value: 'gcash', label: 'GCash', icon: '📱' },
  { value: 'paymaya', label: 'PayMaya/Maya', icon: '📱' },
  { value: 'bank_transfer', label: 'Bank Transfer', icon: '🏦', 
    banks: ['BDO', 'BPI', 'Metrobank', 'Landbank', 'PNB', 'Security Bank', 'UnionBank', 'RCBC', 'EastWest', 'Other'] },
  { value: 'check', label: 'Check', icon: '📝' },
  { value: 'other', label: 'Other', icon: '📦' }
];

// Payment form includes:
// - Amount (PHP)
// - Method (from above)
// - Reference number (GCash ref, bank trace number, check number)
// - Paid date/time
// - Receipt upload (image/PDF)
// - Notes
```

### 7.3 Audit Trail Implementation
```typescript
// Automatic audit logging via database triggers or application middleware
// Every CREATE/UPDATE/DELETE on core tables creates audit_log entry

// Example: Booking status change
{
  entity_type: 'booking',
  entity_id: 'uuid',
  action: 'status_change',
  actor_id: 'admin-uuid',
  actor_role: 'admin',
  old_values: { status: 'pending' },
  new_values: { status: 'confirmed' },
  changed_fields: ['status', 'confirmed_at', 'confirmed_by'],
  metadata: { reason: 'Approved by admin' }
}
```

### 7.4 Notifications (Future Enhancement)
- Email notifications (Supabase Edge Functions + Resend/SendGrid)
- In-app notifications (real-time via Supabase Realtime)
- SMS (optional, via Semaphore/Philippines SMS APIs)

---

## 8. Development Phases

### Phase 1: Foundation (Week 1-2)
- [ ] Project setup (Vite + React + TypeScript + Tailwind)
- [ ] Supabase project setup & database schema
- [ ] Authentication (login, register, password reset, protected routes)
- [ ] Basic layout (sidebar, header, responsive)
- [ ] Role-based route guards
- [ ] API client setup with interceptors

### Phase 2: Core Booking Module (Week 2-3)
- [ ] Resource management (CRUD for admin)
- [ ] Booking creation with availability checking
- [ ] Booking list with filters, pagination, sorting
- [ ] Booking details view
- [ ] Booking status workflow (pending → confirmed → check-in → check-out → completed)
- [ ] Booking cancellation

### Phase 3: User Management & Payments (Week 3-4)
- [ ] User management (admin panel)
- [ ] Role assignment (super_admin only)
- [ ] Payment recording form
- [ ] Payment verification workflow
- [ ] Payment-refund handling
- [ ] Booking-payment linkage

### Phase 4: Audit & History (Week 4)
- [ ] Audit log table & API
- [ ] Audit log viewer with filters
- [ ] Entity-specific history views
- [ ] Export functionality (CSV/PDF)

### Phase 5: Dashboard & Analytics (Week 5)
- [ ] Role-based dashboards
- [ ] Booking statistics
- [ ] Revenue tracking
- [ ] Resource utilization charts
- [ ] Recent activity feed

### Phase 6: Polish & Deploy (Week 5-6)
- [ ] Error boundaries & error handling
- [ ] Loading states & skeleton screens
- [ ] Form validation & user feedback
- [ ] Responsive design testing
- [ ] Accessibility audit
- [ ] CI/CD pipeline setup
- [ ] Production deployment
- [ ] Documentation

---

## 9. Environment Variables

```env
# .env.example
VITE_APP_NAME="Booking Management"
VITE_APP_URL="http://localhost:5173"

# Supabase
VITE_SUPABASE_URL="https://your-project.supabase.co"
VITE_SUPABASE_ANON_KEY="your-anon-key"

# API (if separate backend)
VITE_API_URL="http://localhost:3000/api"

# Feature flags
VITE_ENABLE_REGISTRATION="true"
VITE_ENABLE_EMAIL_NOTIFICATIONS="false"

# Philippines-specific
VITE_DEFAULT_CURRENCY="PHP"
VITE_TIMEZONE="Asia/Manila"
VITE_DATE_FORMAT="MM/DD/YYYY"
```

---

## 10. Security Considerations

1. **Row Level Security (RLS)** - Enforce at database level
2. **JWT Tokens** - Short-lived access tokens, refresh tokens
3. **Input Validation** - Zod schemas on frontend & backend
4. **Rate Limiting** - API rate limiting (Supabase has built-in)
5. **CORS** - Configure properly for production domains
6. **Audit Logs** - Immutable, append-only
7. **Secrets Management** - Environment variables, never commit
8. **Dependency Scanning** - npm audit, Snyk/Dependabot

---

## 11. Testing Strategy

| Type | Tool | Coverage Target |
|------|------|-----------------|
| Unit | Vitest + React Testing Library | 70%+ |
| Integration | Vitest + MSW | Key user flows |
| E2E | Playwright | Critical paths |
| Visual | Storybook + Chromatic | UI components |

---

## 12. Future Enhancements (Post-MVP)

- [ ] Multi-language support (English, Tagalog)
- [ ] Recurring bookings
- [ ] Waitlist management
- [ ] Resource maintenance scheduling
- [ ] Advanced reporting & exports
- [ ] Mobile app (React Native)
- [ ] Webhook integrations
- [ ] Calendar sync (Google, Outlook)
- [ ] SMS notifications (Philippines carriers)
- [ ] QR code check-in
- [ ] Resource packages/bundles
- [ ] Loyalty/points system

---

## 13. Recommended Next Steps

1. **Choose Backend**: Decide on Supabase vs custom backend
2. **Set up Repository**: Initialize Git, create branch strategy
3. **Database Setup**: Run migrations, configure RLS
4. **Start Phase 1**: Begin with project scaffolding

---

## 14. Useful Resources

- [Supabase Docs](https://supabase.com/docs)
- [React Query Docs](https://tanstack.com/query/latest)
- [Zustand Docs](https://github.com/pmndrs/zustand)
- [shadcn/ui Components](https://ui.shadcn.com/)
- [Philippines Bank Codes](https://www.philippinebankcodes.com/)
- [GCash Developer](https://developer.gcash.com/)
- [Maya Business](https://maya.ph/business)

---

*Document Version: 1.0*  
*Created: 2024*  
*Last Updated: 2024*