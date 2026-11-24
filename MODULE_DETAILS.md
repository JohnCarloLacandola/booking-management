# Booking Management System - Detailed Module Breakdown

## Overview
This document provides granular implementation details for each module in the booking management system.

---

## Module 1: Authentication & Authorization (`features/auth`)

### 1.1 Components
```
features/auth/
├── components/
│   ├── LoginForm.tsx              # Email/password + remember me
│   ├── RegisterForm.tsx           # Registration with role selection (admin only)
│   ├── ForgotPasswordForm.tsx     # Email input for reset link
│   ├── ResetPasswordForm.tsx      # New password + confirm (token from URL)
│   ├── AuthLayout.tsx             # Centered card layout for auth pages
│   ├── ProtectedRoute.tsx         # Wrapper for role-based route protection
│   ├── RoleGuard.tsx              # Component-level permission checks
│   └── UserAvatar.tsx             # Dropdown with profile, settings, logout
├── hooks/
│   ├── useAuth.ts                 # Auth state, login, logout, register
│   ├── usePermissions.ts          # Permission checking hooks
│   └── useSession.ts              # Session management, auto-refresh
├── api/
│   └── auth.ts                    # API calls: login, register, refresh, me
├── types/
│   └── auth.ts                    # User, Session, LoginCredentials, RegisterData
├── validators/
│   └── auth.ts                    # Zod schemas for all auth forms
└── utils/
    └── token.ts                   # JWT decode, expiry check, storage
```

### 1.2 Key Features
| Feature | Implementation |
|---------|----------------|
| **Login** | Email/password → JWT access + refresh tokens in httpOnly cookies |
| **Registration** | Invite-only or public (configurable), email verification |
| **Password Reset** | Secure token via email, 1-hour expiry |
| **Session Management** | Auto-refresh 5 min before expiry, logout on 401 |
| **Role Guards** | Route-level (`ProtectedRoute`) + component-level (`RoleGuard`) |
| **Permissions** | `usePermissions()` hook with `can(permission)` function |

### 1.3 Supabase Auth Integration
```typescript
// Auth provider setup
const { data: { session } } = await supabase.auth.getSession()
supabase.auth.onAuthStateChange((event, session) => {
  // Update Zustand store, refresh user profile
})
```

### 1.4 RLS Policies (Database Level)
```sql
-- Users table policies
CREATE POLICY "Users can view own profile" ON users
  FOR SELECT USING (auth.uid() = id OR is_admin());

CREATE POLICY "Admins can manage users" ON users
  FOR ALL USING (is_admin());

-- Function for role checks
CREATE OR REPLACE FUNCTION is_admin() RETURNS BOOLEAN AS $$
  SELECT role IN ('admin', 'super_admin') FROM users WHERE id = auth.uid();
$$ LANGUAGE SQL SECURITY DEFINER;
```

---

## Module 2: Bookings (`features/bookings`)

### 2.1 Components
```
features/bookings/
├── components/
│   ├── BookingList.tsx              # Table with pagination, sorting, filters
│   ├── BookingCard.tsx              # Card view for mobile/calendar
│   ├── BookingForm.tsx              # Create/edit booking (multi-step)
│   ├── BookingDetails.tsx           # Full booking view with timeline
│   ├── BookingCalendar.tsx          # Monthly/weekly calendar view
│   ├── BookingTimeline.tsx          # Visual status progression
│   ├── AvailabilityChecker.tsx      # Real-time slot availability
│   ├── BookingFilters.tsx           # Date range, status, resource, user
│   ├── BookingStatusBadge.tsx       # Colored badge with icon
│   ├── BookingActions.tsx           # Confirm, cancel, check-in/out buttons
│   ├── ResourceSelector.tsx         # Searchable resource dropdown
│   ├── DateTimePicker.tsx           # Custom date/time with timezone
│   └── BookingConflictModal.tsx     # Show conflicts when double-booking
├── hooks/
│   ├── useBookings.ts               # List with filters, pagination
│   ├── useBooking.ts                # Single booking with real-time updates
│   ├── useCreateBooking.ts          # Mutation with optimistic updates
│   ├── useBookingActions.ts         # Confirm, cancel, check-in/out mutations
│   ├── useAvailability.ts           # Check slots for resource/date
│   └── useBookingStats.ts           # Dashboard statistics
├── api/
│   └── bookings.ts                  # All booking API calls
├── types/
│   └── booking.ts                   # Booking, BookingFormData, BookingFilters
├── validators/
│   └── booking.ts                   # Zod schemas for create/update
├── utils/
│   ├── bookingNumber.ts             # Generate BK-YYYY-NNNNNN
│   ├── pricing.ts                   # Calculate amount from resource pricing
│   ├── availability.ts              # Slot conflict detection
│   └── statusTransitions.ts         # Valid status transitions
└── constants/
    └── bookingStatus.ts             # Status labels, colors, icons
```

### 2.2 Booking Form - Multi-Step Flow
```
Step 1: Resource Selection
  - Resource type filter (room, vehicle, equipment, space, service)
  - Searchable dropdown with availability preview
  - Shows capacity, location, pricing

Step 2: Date & Time
  - Date picker (min: today, max: 1 year)
  - Time slots based on resource availability
  - Timezone selector (default: Asia/Manila)
  - Recurring option (future enhancement)

Step 3: Details
  - Booker (self or select user - admin only)
  - Notes / special requests
  - Auto-calculated total amount

Step 4: Review & Submit
  - Summary of all details
  - Terms acceptance
  - Submit → Creates booking with 'pending' status
```

### 2.3 Status Workflow
```
PENDING → CONFIRMED → CHECKED_IN → CHECKED_OUT → COMPLETED
    ↓           ↓           ↓            ↓
CANCELLED   CANCELLED   CANCELLED    CANCELLED
    ↓           ↓           ↓            ↓
  (refund)   (refund)    (partial)    (no refund)

NO_SHOW: Auto-transition if not checked in within 30 min of start
DISPUTED: Manual flag for payment/booking issues
```

### 2.4 Availability Checking Algorithm
```typescript
// Check for overlapping bookings
const hasConflict = (newStart, newEnd, existingBookings) => {
  return existingBookings.some(booking => 
    booking.status !== 'cancelled' &&
    booking.status !== 'no_show' &&
    newStart < booking.end_time &&
    newEnd > booking.start_time
  )
}

// Get available time slots for a resource on a date
const getAvailableSlots = (resourceId, date) => {
  // 1. Get resource operating hours
  // 2. Get existing bookings for date
  // 3. Calculate free slots (30-min increments)
  // 4. Return array of { start, end, available: boolean }
}
```

### 2.5 Philippines-Specific Features
- **Currency**: PHP formatting (₱1,234.56)
- **Timezone**: Asia/Manila (no DST)
- **Date Format**: MM/dd/yyyy
- **Holiday Calendar**: Philippine holidays (configurable)
- **Business Hours**: 8AM-6PM default, per-resource override

---

## Module 3: Resources (`features/resources`)

### 3.1 Components
```
features/resources/
├── components/
│   ├── ResourceList.tsx             # Grid/table with search, filters
│   ├── ResourceCard.tsx             # Card with image, status, pricing
│   ├── ResourceForm.tsx             # Create/edit resource
│   ├── ResourceDetails.tsx          # Full view with booking calendar
│   ├── ResourceImageUpload.tsx      # Drag-drop image upload (Supabase Storage)
│   ├── ResourcePricing.tsx          # Dynamic pricing configuration
│   ├── ResourceAmenities.tsx        # Checklist of amenities
│   └── ResourceStatusBadge.tsx      # Available/Maintenance/Unavailable
├── hooks/
│   ├── useResources.ts              # List with filters
│   ├── useResource.ts               # Single resource
│   ├── useCreateResource.ts         # Mutation
│   └── useResourceBookings.ts       # Bookings for a resource
├── api/
│   └── resources.ts
├── types/
│   └── resource.ts
├── validators/
│   └── resource.ts
└── constants/
    └── resourceTypes.ts             # Types, default amenities, icons
```

### 3.2 Resource Types & Configuration
```typescript
const RESOURCE_TYPES = {
  room: {
    icon: '🏠',
    defaultAmenities: ['wifi', 'ac', 'projector', 'whiteboard'],
    pricingModel: 'hourly|daily',
    capacityRange: [1, 100]
  },
  vehicle: {
    icon: '🚗',
    defaultAmenities: ['ac', 'gps', 'bluetooth'],
    pricingModel: 'hourly|daily|km',
    capacityRange: [1, 50]
  },
  equipment: {
    icon: '🔧',
    defaultAmenities: [],
    pricingModel: 'hourly|daily',
    capacityRange: [1, 1]
  },
  space: {
    icon: '🏟️',
    defaultAmenities: ['wifi', 'ac', 'sound_system'],
    pricingModel: 'hourly|daily|event',
    capacityRange: [10, 1000]
  },
  service: {
    icon: '🛠️',
    defaultAmenities: [],
    pricingModel: 'fixed|hourly',
    capacityRange: [1, 1]
  }
}
```

### 3.3 Pricing Configuration
```typescript
// Flexible pricing JSON structure
interface ResourcePricing {
  model: 'hourly' | 'daily' | 'hourly_daily' | 'fixed' | 'per_km' | 'event';
  currency: 'PHP';
  rates: {
    hourly?: number;        // ₱500/hr
    daily?: number;         // ₱3,000/day
    fixed?: number;         // ₱1,500 flat
    perKm?: number;         // ₱15/km
    event?: number;         // ₱50,000/event
    overtime?: number;      // ₱750/hr after hours
  };
  discounts?: {
    weekly?: number;        // 10% off 7+ days
    monthly?: number;       // 20% off 30+ days
    member?: number;        // 5% for registered users
  };
  minimumHours?: number;    // 2 hours minimum
  maximumHours?: number;    // 24 hours maximum
}
```

---

## Module 4: Users (`features/users`)

### 4.1 Components
```
features/users/
├── components/
│   ├── UserList.tsx                 # Admin table: search, filter, paginate
│   ├── UserCard.tsx                 # Compact view for assignment
│   ├── UserForm.tsx                 # Create/edit user (admin)
│   ├── UserDetails.tsx              # Profile + activity + bookings
│   ├── UserRoleSelector.tsx         # Role dropdown with permissions preview
│   ├── UserStatusBadge.tsx          # Active/Inactive/Suspended/Pending
│   ├── UserAvatar.tsx               # Upload/remove avatar
│   ├── UserFilters.tsx              # Role, status, date joined
│   └── BulkActions.tsx              # Activate, suspend, delete, export
├── hooks/
│   ├── useUsers.ts                  # List with filters
│   ├── useUser.ts                   # Single user
│   ├── useCreateUser.ts             # Mutation (invite or direct)
│   ├── useUpdateUser.ts             # Profile, role, status updates
│   └── useUserStats.ts              # Dashboard stats
├── api/
│   └── users.ts
├── types/
│   └── user.ts
├── validators/
│   └── user.ts
└── constants/
    └── userRoles.ts                 # Role definitions, permissions
```

### 4.2 User Management Features
| Feature | Super Admin | Admin | User |
|---------|-------------|-------|------|
| Create user | ✅ (any role) | ✅ (user only) | ❌ |
| Edit user | ✅ | ✅ (not super_admin) | Own profile |
| Change role | ✅ | ❌ | ❌ |
| Suspend/Activate | ✅ | ✅ | ❌ |
| Delete user | ✅ (soft) | ❌ | ❌ |
| View all users | ✅ | ✅ | ❌ |
| Reset password | ✅ | ✅ | Own |

### 4.3 Invitation Flow (Admin Creates User)
```
1. Admin fills: email, full_name, role, optional phone
2. System sends invitation email with magic link
3. User clicks link → sets password → account activated
4. Auto-login after password set
5. Audit log: user_create action
```

---

## Module 5: Payments (`features/payments`)

### 5.1 Components
```
features/payments/
├── components/
│   ├── PaymentList.tsx              # Table with filters, pagination
│   ├── PaymentForm.tsx              # Record payment (multi-step)
│   ├── PaymentDetails.tsx           # Full view with receipt
│   ├── PaymentMethodSelector.tsx    # Philippines-specific methods
│   ├── BankSelector.tsx             # Philippine banks dropdown
│   ├── ReceiptUpload.tsx            # Drag-drop image/PDF upload
│   ├── PaymentStatusBadge.tsx       # Pending/Verified/Partial/Refunded
│   ├── PaymentActions.tsx           # Verify, refund, edit
│   ├── PaymentFilters.tsx           # Date, method, status, booking
│   └── PaymentSummary.tsx           # Booking payment progress bar
├── hooks/
│   ├── usePayments.ts               # List with filters
│   ├── usePayment.ts                # Single payment
│   ├── useCreatePayment.ts          # Mutation
│   ├── useVerifyPayment.ts          # Admin verification
│   ├── useRefundPayment.ts          # Refund workflow
│   └── usePaymentStats.ts           # Revenue, pending, verified
├── api/
│   └── payments.ts
├── types/
│   └── payment.ts
├── validators/
│   └── payment.ts
├── utils/
│   ├── paymentNumber.ts             # Generate PAY-YYYY-NNNNNN
│   ├── currency.ts                  # PHP formatting
│   └── receipt.ts                   # Receipt validation, storage
└── constants/
    └── paymentMethods.ts            # PH methods, banks, icons
```

### 5.2 Philippines Payment Methods Detail
```typescript
const PH_PAYMENT_METHODS = [
  {
    value: 'cash',
    label: 'Cash',
    icon: '💵',
    fields: ['amount', 'paid_at', 'notes'],
    requiresReference: false,
    requiresReceipt: false
  },
  {
    value: 'gcash',
    label: 'GCash',
    icon: '📱',
    fields: ['amount', 'paid_at', 'reference_number', 'receipt_url', 'notes'],
    requiresReference: true,
    referenceLabel: 'GCash Reference #',
    referencePattern: '^\\d{13}$',  // 13 digits
    requiresReceipt: true
  },
  {
    value: 'paymaya',
    label: 'PayMaya / Maya',
    icon: '📱',
    fields: ['amount', 'paid_at', 'reference_number', 'receipt_url', 'notes'],
    requiresReference: true,
    referenceLabel: 'Maya Transaction ID',
    referencePattern: '^\\w{20,30}$',
    requiresReceipt: true
  },
  {
    value: 'bank_transfer',
    label: 'Bank Transfer',
    icon: '🏦',
    fields: ['amount', 'paid_at', 'reference_number', 'bank_name', 'receipt_url', 'notes'],
    requiresReference: true,
    referenceLabel: 'Bank Reference / Trace #',
    requiresReceipt: true,
    banks: [
      'BDO', 'BPI', 'Metrobank', 'Landbank', 'PNB',
      'Security Bank', 'UnionBank', 'RCBC', 'EastWest',
      'China Bank', 'DBP', 'PSBank', 'RCBC', 'Other'
    ]
  },
  {
    value: 'check',
    label: 'Check',
    icon: '📝',
    fields: ['amount', 'paid_at', 'reference_number', 'bank_name', 'check_date', 'notes'],
    requiresReference: true,
    referenceLabel: 'Check Number',
    requiresReceipt: false
  },
  {
    value: 'other',
    label: 'Other',
    icon: '📦',
    fields: ['amount', 'paid_at', 'reference_number', 'notes'],
    requiresReference: false,
    requiresReceipt: false
  }
]
```

### 5.3 Payment Verification Workflow
```
1. User records payment → Status: PENDING
2. Admin reviews: receipt, reference number, amount match
3. Admin clicks "Verify" → Status: VERIFIED
   - Updates booking payment progress
   - Creates audit log: payment_verify
4. If partial: Status: PARTIAL (multiple payments for one booking)
5. Refund: Admin initiates → Status: REFUNDED
   - Records refund amount, reason
   - Updates booking payment progress
   - Creates audit log: payment_refund
```

### 5.4 Payment-Booking Linkage
```typescript
// Booking payment status computed from payments
interface BookingPaymentStatus {
  totalAmount: number;           // Booking total
  paidAmount: number;            // Sum of verified payments
  pendingAmount: number;         // Sum of pending payments
  refundedAmount: number;        // Sum of refunded
  balance: number;               // total - paid + refunded
  status: 'unpaid' | 'partial' | 'paid' | 'overpaid' | 'refunded';
  payments: Payment[];           // All payments for this booking
}
```

---

## Module 6: Audit Logs (`features/audit`)

### 6.1 Components
```
features/audit/
├── components/
│   ├── AuditLogList.tsx             # Table with advanced filters
│   ├── AuditLogDetails.tsx          # Expanded view with diff
│   ├── AuditLogFilters.tsx          # Entity, action, actor, date range
│   ├── EntityHistory.tsx            # Timeline for specific entity
│   ├── AuditDiffViewer.tsx          # Side-by-side old/new values
│   ├── AuditExport.tsx              # CSV/PDF export with filters
│   └── AuditStats.tsx               # Activity summary cards
├── hooks/
│   ├── useAuditLogs.ts              # List with filters, pagination
│   ├── useAuditLog.ts               # Single log entry
│   ├── useEntityHistory.ts          # History for specific entity
│   └── useAuditExport.ts            # Export mutation
├── api/
│   └── audit.ts
├── types/
│   └── audit.ts
├── utils/
│   ├── diff.ts                      # Compute field-level diffs
│   ├── export.ts                    # CSV/PDF generation
│   └── filters.ts                   # Filter builders
└── constants/
    └── auditActions.ts              # Action labels, categories, icons
```

### 6.2 Audit Action Categories
```typescript
const AUDIT_CATEGORIES = {
  booking: [
    'create', 'update', 'delete', 'restore',
    'status_change', 'confirm', 'cancel', 'checkin', 'checkout'
  ],
  user: [
    'create', 'update', 'delete', 'restore',
    'role_change', 'status_change', 'password_change', 'login', 'logout'
  ],
  resource: [
    'create', 'update', 'delete', 'restore', 'archive'
  ],
  payment: [
    'create', 'update', 'delete', 'verify', 'refund'
  ],
  settings: [
    'update', 'security_change'
  ],
  system: [
    'bulk_action', 'export', 'import'
  ]
}
```

### 6.3 Diff Viewer Implementation
```typescript
// Show field-level changes with color coding
interface FieldDiff {
  field: string;
  label: string;
  oldValue: any;
  newValue: any;
  type: 'added' | 'removed' | 'changed' | 'unchanged';
}

// Render as:
// - Green background for added
// - Red background for removed  
// - Yellow background for changed
// - Gray for unchanged (collapsed by default)
```

### 6.4 Export Features
- **CSV**: All filtered logs, flattened JSON fields
- **PDF**: Formatted report with summary + details
- **Date Range**: Preset (today, week, month, custom)
- **Filters Applied**: Included in export metadata

---

## Module 7: Dashboard (`features/dashboard`)

### 7.1 Components
```
features/dashboard/
├── components/
│   ├── DashboardLayout.tsx          # Role-based layout wrapper
│   ├── StatsCards.tsx               # KPI cards (bookings, revenue, etc.)
│   ├── BookingTrendChart.tsx        # Line/bar chart (Recharts)
│   ├── RevenueChart.tsx             # Revenue by period
│   ├── ResourceUtilizationChart.tsx # Heatmap or bar chart
│   ├── RecentActivity.tsx           # Latest bookings, payments, users
│   ├── UpcomingBookings.tsx         # Next 7 days bookings
│   ├── QuickActions.tsx             # Role-based action buttons
│   ├── UserBookingSummary.tsx       # User's own bookings (user role)
│   └── AdminOverview.tsx            # System-wide stats (admin+)
├── hooks/
│   ├── useDashboardStats.ts         # Role-based stats
│   ├── useBookingTrends.ts          # Time-series data
│   ├── useRevenueData.ts            # Revenue analytics
│   ├── useResourceUtilization.ts    # Utilization percentages
│   └── useRecentActivity.ts         # Combined activity feed
├── api/
│   └── dashboard.ts
├── types/
│   └── dashboard.ts
└── utils/
    └── chartConfig.ts               # Recharts configurations
```

### 7.2 Role-Based Dashboard Content

#### Super Admin Dashboard
```
┌─────────────────────────────────────────────────────────────┐
│ Stats Cards: Total Users | Total Resources | Total Bookings │
│              Revenue (Month) | Pending Payments | Audit Logs│
├─────────────────────────────────────────────────────────────┤
│ Booking Trends (30 days)     │ Revenue by Payment Method    │
├─────────────────────────────────────────────────────────────┤
│ Resource Utilization Heatmap │ Recent System Activity       │
├─────────────────────────────────────────────────────────────┤
│ Quick Actions: [Add User] [Add Resource] [View Audit]      │
└─────────────────────────────────────────────────────────────┘
```

#### Admin Dashboard
```
┌─────────────────────────────────────────────────────────────┐
│ Stats Cards: My Bookings | Pending Confirmations | Revenue │
│              Resources Managed | Pending Payments           │
├─────────────────────────────────────────────────────────────┤
│ Booking Trends (30 days)     │ Resource Utilization         │
├─────────────────────────────────────────────────────────────┤
│ Pending Bookings to Confirm  │ Recent Payments to Verify    │
├─────────────────────────────────────────────────────────────┤
│ Quick Actions: [New Booking] [Manage Resources] [Users]    │
└─────────────────────────────────────────────────────────────┘
```

#### User Dashboard
```
┌─────────────────────────────────────────────────────────────┐
│ Stats Cards: My Bookings | Upcoming | Past | Spent (PHP)   │
├─────────────────────────────────────────────────────────────┤
│ My Upcoming Bookings (Next 7 Days)                         │
├─────────────────────────────────────────────────────────────┤
│ My Booking History (Last 30 Days)                          │
├─────────────────────────────────────────────────────────────┤
│ Quick Actions: [New Booking] [View Resources] [Payments]   │
└─────────────────────────────────────────────────────────────┘
```

---

## Module 8: Settings (`features/settings`)

### 8.1 Components
```
features/settings/
├── components/
│   ├── SettingsLayout.tsx           # Tabbed layout
│   ├── GeneralSettings.tsx          # App name, timezone, currency, date format
│   ├── SecuritySettings.tsx         # RLS, password policy, session timeout (super_admin)
│   ├── NotificationSettings.tsx     # Email, SMS, in-app preferences
│   ├── PaymentSettings.tsx          # Enabled methods, bank list, receipt required
│   ├── ResourceSettings.tsx         # Default types, amenities, pricing templates
│   ├── BookingSettings.tsx          # Min/max advance, cancellation policy, slots
│   ├── UserSettings.tsx             # Registration, roles, default role
│   ├── AppearanceSettings.tsx       # Theme, logo, colors
│   └── BackupSettings.tsx           # Export/import data (super_admin)
├── hooks/
│   ├── useSettings.ts               # Fetch/update settings
│   └── useSettingsSections.ts       # Section visibility by role
├── api/
│   └── settings.ts
├── types/
│   └── settings.ts
├── validators/
│   └── settings.ts
└── constants/
    └── defaultSettings.ts           # Default values for all settings
```

### 8.2 Settings by Role Access
| Section | Super Admin | Admin | User |
|---------|-------------|-------|------|
| General | ✅ | ✅ | ❌ |
| Security | ✅ | ❌ | ❌ |
| Notifications | ✅ | ✅ | Own only |
| Payments | ✅ | ✅ | ❌ |
| Resources | ✅ | ✅ | ❌ |
| Bookings | ✅ | ✅ | ❌ |
| Users | ✅ | Limited | ❌ |
| Appearance | ✅ | ❌ | ❌ |
| Backup | ✅ | ❌ | ❌ |

---

## Module 9: Shared Components (`shared/components`)

### 9.1 UI Primitives (shadcn/ui style)
```
shared/components/ui/
├── button.tsx
├── input.tsx
├── textarea.tsx
├── select.tsx
├── checkbox.tsx
├── radio-group.tsx
├── switch.tsx
├── label.tsx
├── card.tsx
├── table.tsx
├── pagination.tsx
├── badge.tsx
├── avatar.tsx
├── dropdown-menu.tsx
├── dialog.tsx
├── alert-dialog.tsx
├── toast.tsx
├── tooltip.tsx
├── popover.tsx
├── tabs.tsx
├── accordion.tsx
├── separator.tsx
├── scroll-area.tsx
├── progress.tsx
├── slider.tsx
├── skeleton.tsx
├── hover-card.tsx
├── context-menu.tsx
├── menubar.tsx
├── navigation-menu.tsx
├── command.tsx
├── calendar.tsx
├── date-picker.tsx
├── form.tsx
├── sheet.tsx
├── drawer.tsx
└── collapsible.tsx
```

### 9.2 Layout Components
```
shared/components/layout/
├── Sidebar.tsx              # Collapsible, role-based navigation
├── Header.tsx               # User menu, notifications, theme toggle
├── Footer.tsx               # Version, links
├── PageContainer.tsx        # Consistent page padding, max-width
├── Section.tsx              # Section with title, description, actions
├── Grid.tsx                 # Responsive grid wrapper
├── Flex.tsx                 # Flexbox wrapper
└── Container.tsx            # Max-width container
```

### 9.3 Data Display Components
```
shared/components/data-display/
├── DataTable.tsx            # Sortable, filterable, paginated table
├── DataCard.tsx             # Card with consistent structure
├── StatCard.tsx             # KPI card with trend indicator
├── ProgressCard.tsx         # Progress bar with label
├── Timeline.tsx             # Vertical timeline for history
├── EmptyState.tsx           # Illustrations for empty lists
├── LoadingState.tsx         # Skeletons, spinners
├── ErrorState.tsx           # Error with retry action
└── AvatarGroup.tsx          # Stacked avatars with count
```

### 9.4 Form Components
```
shared/components/forms/
├── FormField.tsx            # Label + input + error + hint
├── FormSection.tsx          # Grouped fields with title
├── FileUpload.tsx           # Drag-drop with preview
├── ImageUpload.tsx          # Image-specific with crop
├── RichTextEditor.tsx       # TipTap or similar
├── AddressInput.tsx         # Philippines address autocomplete
├── PhoneInput.tsx           # +63 format with validation
├── CurrencyInput.tsx        # PHP formatting
├── DateRangePicker.tsx      # Start/end date
└── MultiSelect.tsx          # Searchable multi-select
```

---

## Module 10: Shared Hooks (`shared/hooks`)

### 10.1 Core Hooks
```typescript
// shared/hooks/index.ts
export { useAuth } from './useAuth'
export { usePermissions } from './usePermissions'
export { useQuery, useMutation, useQueryClient } from '@tanstack/react-query'
export { useStore } from './useStore'           // Zustand
export { useDebounce } from './useDebounce'
export { useLocalStorage } from './useLocalStorage'
export { useMediaQuery } from './useMediaQuery'
export { useOnClickOutside } from './useOnClickOutside'
export { useKeyboardShortcut } from './useKeyboardShortcut'
export { useCopyToClipboard } from './useCopyToClipboard'
export { useFileUpload } from './useFileUpload'
export { useImagePreview } from './useImagePreview'
```

### 10.2 Feature-Specific Hooks (Re-exported)
```typescript
// Bookings
export { useBookings, useBooking, useCreateBooking, useBookingActions } from '@/features/bookings/hooks'

// Resources
export { useResources, useResource } from '@/features/resources/hooks'

// Users
export { useUsers, useUser } from '@/features/users/hooks'

// Payments
export { usePayments, usePayment, useCreatePayment, useVerifyPayment } from '@/features/payments/hooks'

// Audit
export { useAuditLogs, useEntityHistory } from '@/features/audit/hooks'

// Dashboard
export { useDashboardStats, useBookingTrends } from '@/features/dashboard/hooks'
```

---

## Module 11: Shared Utils (`shared/utils`)

### 11.1 Utility Functions
```
shared/utils/
├── format.ts                # Currency, date, phone, number formatting
├── validation.ts            # Common validators (email, phone, etc.)
├── string.ts                # Slugify, truncate, capitalize
├── date.ts                  # Date helpers (addDays, isWeekend, etc.)
├── array.ts                 # GroupBy, sortBy, unique, chunk
├── object.ts                # Pick, omit, deepClone, diff
├── url.ts                   # Query string builders
├── storage.ts               # LocalStorage with type safety
├── error.ts                 # Error formatting, user-friendly messages
├── constants.ts             # App-wide constants
├── helpers.ts               # Misc helpers (generateId, classNames)
└── philippines.ts           # PH-specific: holidays, banks, regions
```

### 11.2 Philippines-Specific Utilities
```typescript
// shared/utils/philippines.ts

export const PH_HOLIDAYS_2024 = [
  '2024-01-01', // New Year's Day
  '2024-03-28', // Maundy Thursday
  '2024-03-29', // Good Friday
  '2024-04-09', // Araw ng Kagitingan
  '2024-05-01', // Labor Day
  '2024-06-12', // Independence Day
  '2024-08-26', // National Heroes Day
  '2024-11-30', // Bonifacio Day
  '2024-12-25', // Christmas Day
  '2024-12-30', // Rizal Day
  // ... movable holidays
]

export const PH_BANKS = [
  { code: 'BDO', name: 'BDO Unibank', swift: 'BNORPHMM' },
  { code: 'BPI', name: 'Bank of the Philippine Islands', swift: 'BOPIPHMM' },
  { code: 'MBTC', name: 'Metrobank', swift: 'MBTCPHMM' },
  // ... all major banks
]

export const PH_REGIONS = [
  { code: 'NCR', name: 'National Capital Region' },
  { code: 'CAR', name: 'Cordillera Administrative Region' },
  { code: 'I', name: 'Ilocos Region' },
  // ... all 17 regions
]

export function formatPHP(amount: number): string {
  return new Intl.NumberFormat('en-PH', {
    style: 'currency',
    currency: 'PHP',
    minimumFractionDigits: 2
  }).format(amount)
}

export function formatPHDate(date: Date | string): string {
  return new Intl.DateTimeFormat('en-PH', {
    year: 'numeric',
    month: '2-digit',
    day: '2-digit'
  }).format(new Date(date))
}

export function formatPHDateTime(date: Date | string): string {
  return new Intl.DateTimeFormat('en-PH', {
    year: 'numeric',
    month: '2-digit',
    day: '2-digit',
    hour: '2-digit',
    minute: '2-digit',
    hour12: true
  }).format(new Date(date))
}
```

---

## Module 12: API Layer (`shared/api`)

### 12.1 Structure
```
shared/api/
├── client.ts                # Axios instance with interceptors
├── endpoints.ts             # API endpoint constants
├── queries.ts               # TanStack Query keys factory
├── mutations.ts             # Common mutation helpers
├── errors.ts                # Error handling, retry logic
├── auth.ts                  # Auth header management
└── supabase.ts              # Supabase client (if used directly)
```

### 12.2 Axios Client Configuration
```typescript
// shared/api/client.ts
import axios from 'axios'
import { getAccessToken, refreshAccessToken } from '@/features/auth/utils/token'

export const apiClient = axios.create({
  baseURL: import.meta.env.VITE_API_URL || '/api',
  timeout: 30000,
  headers: {
    'Content-Type': 'application/json'
  }
})

// Request interceptor - add auth token
apiClient.interceptors.request.use(async (config) => {
  const token = getAccessToken()
  if (token) {
    config.headers.Authorization = `Bearer ${token}`
  }
  return config
})

// Response interceptor - handle 401, refresh token
apiClient.interceptors.response.use(
  (response) => response,
  async (error) => {
    const originalRequest = error.config
    
    if (error.response?.status === 401 && !originalRequest._retry) {
      originalRequest._retry = true
      
      try {
        const newToken = await refreshAccessToken()
        originalRequest.headers.Authorization = `Bearer ${newToken}`
        return apiClient(originalRequest)
      } catch {
        // Refresh failed - logout
        window.location.href = '/login'
      }
    }
    
    return Promise.reject(error)
  }
)
```

### 12.3 Query Keys Factory
```typescript
// shared/api/queries.ts
export const queryKeys = {
  auth: {
    me: () => ['auth', 'me'] as const,
    session: () => ['auth', 'session'] as const,
  },
  bookings: {
    all: (filters?: BookingFilters) => ['bookings', 'list', filters] as const,
    detail: (id: string) => ['bookings', 'detail', id] as const,
    availability: (resourceId: string, date: string) => 
      ['bookings', 'availability', resourceId, date] as const,
    stats: (role: Role) => ['bookings', 'stats', role] as const,
  },
  resources: {
    all: (filters?: ResourceFilters) => ['resources', 'list', filters] as const,
    detail: (id: string) => ['resources', 'detail', id] as const,
    bookings: (id: string) => ['resources', id, 'bookings'] as const,
  },
  users: {
    all: (filters?: UserFilters) => ['users', 'list', filters] as const,
    detail: (id: string) => ['users', 'detail', id] as const,
  },
  payments: {
    all: (filters?: PaymentFilters) => ['payments', 'list', filters] as const,
    detail: (id: string) => ['payments', 'detail', id] as const,
    byBooking: (bookingId: string) => ['payments', 'booking', bookingId] as const,
  },
  audit: {
    all: (filters?: AuditFilters) => ['audit', 'list', filters] as const,
    detail: (id: string) => ['audit', 'detail', id] as const,
    entityHistory: (entityType: string, entityId: string) => 
      ['audit', 'entity', entityType, entityId] as const,
  },
  dashboard: {
    stats: (role: Role) => ['dashboard', 'stats', role] as const,
    trends: (period: string) => ['dashboard', 'trends', period] as const,
    revenue: (period: string) => ['dashboard', 'revenue', period] as const,
  },
  settings: {
    all: () => ['settings'] as const,
    section: (section: string) => ['settings', section] as const,
  }
} as const
```

---

## Module 13: Validators (`shared/validators`)

### 13.1 Common Validators
```typescript
// shared/validators/common.ts
import { z } from 'zod'

export const emailSchema = z.string().email('Invalid email address')
export const phoneSchema = z.string().regex(/^\+63\d{10}$/, 'Invalid PH phone number (+63XXXXXXXXXX)')
export const passwordSchema = z.string().min(8, 'Password must be at least 8 characters')
export const requiredString = (field: string) => z.string().min(1, `${field} is required`)
export const positiveNumber = (field: string) => z.number().positive(`${field} must be positive`)
export const futureDate = (field: string) => z.date().refine(d => d > new Date(), `${field} must be in the future`)
export const pastDate = (field: string) => z.date().refine(d => d < new Date(), `${field} must be in the past`)
```

### 13.2 Philippines-Specific Validators
```typescript
// shared/validators/philippines.ts
import { z } from 'zod'

export const phPhoneSchema = z.string().regex(
  /^(\+63|0)9\d{9}$/,
  'Invalid Philippine mobile number (e.g., +639171234567 or 09171234567)'
)

export const phLandlineSchema = z.string().regex(
  /^(\+63|0)(2|[3-9])\d{7,8}$/,
  'Invalid Philippine landline number'
)

export const gcashRefSchema = z.string().regex(
  /^\d{13}$/,
  'GCash reference must be 13 digits'
)

export const mayaRefSchema = z.string().regex(
  /^[A-Z0-9]{20,30}$/i,
  'Invalid Maya transaction ID'
)

export const bankRefSchema = z.string().min(1, 'Bank reference is required')

export const checkNumberSchema = z.string().regex(
  /^\d{6,10}$/,
  'Check number must be 6-10 digits'
)

export const phZipCodeSchema = z.string().regex(
  /^\d{4}$/,
  'Philippine ZIP code must be 4 digits'
)
```

---

## Module 14: Constants (`shared/constants`)

### 14.1 Application Constants
```typescript
// shared/constants/app.ts
export const APP_NAME = import.meta.env.VITE_APP_NAME || 'Booking Management'
export const APP_VERSION = '1.0.0'
export const DEFAULT_PAGE_SIZE = 10
export const MAX_PAGE_SIZE = 100
export const DEBOUNCE_MS = 300
export const TOAST_DURATION = 4000
export const SESSION_WARNING_MINUTES = 5
```

### 14.2 Role & Permission Constants
```typescript
// shared/constants/roles.ts
export const ROLES = {
  SUPER_ADMIN: 'super_admin',
  ADMIN: 'admin',
  USER: 'user'
} as const

export const ROLE_LABELS = {
  super_admin: 'Super Admin',
  admin: 'Admin',
  user: 'User'
}

export const ROLE_HIERARCHY = ['super_admin', 'admin', 'user']

export const PERMISSIONS = {
  // Bookings
  BOOKINGS_CREATE: 'bookings:create',
  BOOKINGS_READ: 'bookings:read',
  BOOKINGS_READ_OWN: 'bookings:read:own',
  BOOKINGS_UPDATE: 'bookings:update',
  BOOKINGS_DELETE: 'bookings:delete',
  BOOKINGS_CONFIRM: 'bookings:confirm',
  BOOKINGS_CANCEL: 'bookings:cancel',
  BOOKINGS_CANCEL_OWN: 'bookings:cancel:own',
  BOOKINGS_CHECKIN: 'bookings:checkin',
  BOOKINGS_CHECKOUT: 'bookings:checkout',
  
  // Resources
  RESOURCES_CREATE: 'resources:create',
  RESOURCES_READ: 'resources:read',
  RESOURCES_READ_AVAILABLE: 'resources:read:available',
  RESOURCES_UPDATE: 'resources:update',
  RESOURCES_DELETE: 'resources:delete',
  
  // Users
  USERS_CREATE: 'users:create',
  USERS_READ: 'users:read',
  USERS_UPDATE: 'users:update',
  USERS_DELETE: 'users:delete',
  USERS_MANAGE_ROLES: 'users:manage_roles',
  
  // Payments
  PAYMENTS_CREATE: 'payments:create',
  PAYMENTS_CREATE_OWN: 'payments:create:own',
  PAYMENTS_READ: 'payments:read',
  PAYMENTS_READ_OWN: 'payments:read:own',
  PAYMENTS_VERIFY: 'payments:verify',
  PAYMENTS_REFUND: 'payments:refund',
  
  // Audit
  AUDIT_READ: 'audit:read',
  AUDIT_READ_SECURITY: 'audit:read_security',
  AUDIT_READ_OWN: 'audit:read:own',
  AUDIT_EXPORT: 'audit:export',
  
  // Settings
  SETTINGS_READ: 'settings:read',
  SETTINGS_UPDATE: 'settings:update',
  SETTINGS_SECURITY: 'settings:security',
} as const
```

### 14.3 Status Constants
```typescript
// shared/constants/status.ts
export const BOOKING_STATUS = {
  PENDING: 'pending',
  CONFIRMED: 'confirmed',
  CHECKED_IN: 'checked_in',
  CHECKED_OUT: 'checked_out',
  COMPLETED: 'completed',
  CANCELLED: 'cancelled',
  NO_SHOW: 'no_show',
  DISPUTED: 'disputed'
} as const

export const BOOKING_STATUS_LABELS = {
  pending: 'Pending',
  confirmed: 'Confirmed',
  checked_in: 'Checked In',
  checked_out: 'Checked Out',
  completed: 'Completed',
  cancelled: 'Cancelled',
  no_show: 'No Show',
  disputed: 'Disputed'
}

export const BOOKING_STATUS_COLORS = {
  pending: 'warning',
  confirmed: 'info',
  checked_in: 'primary',
  checked_out: 'secondary',
  completed: 'success',
  cancelled: 'destructive',
  no_show: 'destructive',
  disputed: 'warning'
}

export const PAYMENT_STATUS = {
  PENDING: 'pending',
  VERIFIED: 'verified',
  PARTIAL: 'partial',
  REFUNDED: 'refunded',
  FAILED: 'failed',
  DISPUTED: 'disputed'
} as const

export const RESOURCE_STATUS = {
  AVAILABLE: 'available',
  MAINTENANCE: 'maintenance',
  UNAVAILABLE: 'unavailable',
  ARCHIVED: 'archived'
} as const

export const USER_STATUS = {
  ACTIVE: 'active',
  INACTIVE: 'inactive',
  SUSPENDED: 'suspended',
  PENDING: 'pending'
} as const
```

---

## Module 15: Types (`shared/types`)

### 15.1 Core Type Definitions
```typescript
// shared/types/index.ts

// Base types
export interface BaseEntity {
  id: string
  created_at: string
  updated_at: string
  deleted_at?: string
}

export interface PaginatedResponse<T> {
  data: T[]
  meta: {
    total: number
    page: number
    page_size: number
    total_pages: number
  }
}

export interface ApiError {
  message: string
  code?: string
  details?: Record<string, string[]>
  status: number
}

// User types
export interface User extends BaseEntity {
  email: string
  full_name: string | null
  phone: string | null
  avatar_url: string | null
  role: Role
  status: UserStatus
  email_verified_at: string | null
  last_login_at: string | null
}

export type Role = 'super_admin' | 'admin' | 'user'
export type UserStatus = 'active' | 'inactive' | 'suspended' | 'pending'

// Resource types
export interface Resource extends BaseEntity {
  name: string
  description: string | null
  type: ResourceType
  capacity: number
  location: string | null
  amenities: string[]
  pricing: ResourcePricing
  status: ResourceStatus
  created_by: string
  images: string[]
}

export type ResourceType = 'room' | 'vehicle' | 'equipment' | 'space' | 'service'
export type ResourceStatus = 'available' | 'maintenance' | 'unavailable' | 'archived'

export interface ResourcePricing {
  model: 'hourly' | 'daily' | 'hourly_daily' | 'fixed' | 'per_km' | 'event'
  currency: 'PHP'
  rates: {
    hourly?: number
    daily?: number
    fixed?: number
    per_km?: number
    event?: number
    overtime?: number
  }
  discounts?: {
    weekly?: number
    monthly?: number
    member?: number
  }
  minimum_hours?: number
  maximum_hours?: number
}

// Booking types
export interface Booking extends BaseEntity {
  booking_number: string
  resource_id: string
  user_id: string
  booker_id: string | null
  status: BookingStatus
  start_time: string
  end_time: string
  timezone: string
  total_amount: number
  currency: 'PHP'
  notes: string | null
  cancellation_reason: string | null
  cancelled_at: string | null
  cancelled_by: string | null
  confirmed_at: string | null
  confirmed_by: string | null
  checked_in_at: string | null
  checked_out_at: string | null
  resource?: Resource
  user?: User
  booker?: User
  payments?: Payment[]
}

export type BookingStatus = 
  | 'pending' | 'confirmed' | 'checked_in' | 'checked_out'
  | 'completed' | 'cancelled' | 'no_show' | 'disputed'

// Payment types
export interface Payment extends BaseEntity {
  payment_number: string
  booking_id: string
  user_id: string
  amount: number
  currency: 'PHP'
  method: PaymentMethod
  reference_number: string | null
  status: PaymentStatus
  paid_at: string
  notes: string | null
  receipt_url: string | null
  verified_at: string | null
  verified_by: string | null
  refunded_at: string | null
  refunded_amount: number | null
  refund_reason: string | null
  booking?: Booking
  user?: User
}

export type PaymentMethod = 
  | 'cash' | 'gcash' | 'paymaya' | 'bank_transfer' 
  | 'credit_card' | 'debit_card' | 'check' | 'other'

export type PaymentStatus = 
  | 'pending' | 'verified' | 'partial' | 'refunded' | 'failed' | 'disputed'

// Audit types
export interface AuditLog extends BaseEntity {
  entity_type: string
  entity_id: string
  action: AuditAction
  actor_id: string | null
  actor_role: Role | null
  old_values: Record<string, any> | null
  new_values: Record<string, any> | null
  changed_fields: string[]
  ip_address: string | null
  user_agent: string | null
  metadata: Record<string, any>
  actor?: User
}

export type AuditAction = 
  | 'create' | 'update' | 'delete' | 'restore'
  | 'status_change' | 'assign' | 'unassign'
  | 'login' | 'logout' | 'password_change' | 'role_change'
  | 'payment_record' | 'payment_verify' | 'payment_refund'
  | 'booking_confirm' | 'booking_cancel' | 'booking_checkin' | 'booking_checkout'
  | 'resource_create' | 'resource_update' | 'resource_archive'
  | 'user_create' | 'user_update' | 'user_suspend' | 'user_activate'
  | 'settings_change' | 'bulk_action' | 'export' | 'import'

// Form types
export interface BookingFormData {
  resource_id: string
  booker_id?: string
  start_time: Date
  end_time: Date
  timezone: string
  notes?: string
}

export interface PaymentFormData {
  booking_id: string
  amount: number
  method: PaymentMethod
  reference_number?: string
  bank_name?: string
  paid_at: Date
  notes?: string
  receipt_file?: File
}

export interface ResourceFormData {
  name: string
  description?: string
  type: ResourceType
  capacity: number
  location?: string
  amenities: string[]
  pricing: ResourcePricing
  status: ResourceStatus
  images?: File[]
}

export interface UserFormData {
  email: string
  full_name: string
  phone?: string
  role: Role
  status: UserStatus
  password?: string // Only for create
}
```

---

## Module 16: Database Migrations (`supabase/migrations`)

### 16.1 Migration Files
```
supabase/migrations/
├── 20240101000001_initial_schema.sql      # Core tables
├── 20240101000002_rls_policies.sql        # Row Level Security
├── 20240101000003_indexes.sql             # Performance indexes
├── 20240101000004_functions.sql           # Helper functions
├── 20240101000005_triggers.sql            # Audit triggers
├── 20240101000006_seed_data.sql           # Default data
└── 20240101000007_storage_buckets.sql     # File storage setup
```

### 16.2 Audit Trigger Example
```sql
-- supabase/migrations/20240101000005_triggers.sql

-- Function to create audit log entries
CREATE OR REPLACE FUNCTION create_audit_log()
RETURNS TRIGGER AS $$
DECLARE
  v_old_values JSONB;
  v_new_values JSONB;
  v_changed_fields TEXT[];
  v_actor_id UUID;
  v_actor_role USER_ROLE;
BEGIN
  -- Get current user from auth context
  v_actor_id := auth.uid();
  SELECT role INTO v_actor_role FROM users WHERE id = v_actor_id;
  
  -- Determine changed fields
  IF TG_OP = 'INSERT' THEN
    v_new_values := to_jsonb(NEW);
    v_old_values := NULL;
    v_changed_fields := ARRAY(SELECT jsonb_object_keys(to_jsonb(NEW)));
  ELSIF TG_OP = 'UPDATE' THEN
    v_old_values := to_jsonb(OLD);
    v_new_values := to_jsonb(NEW);
    v_changed_fields := ARRAY(
      SELECT key FROM jsonb_each(to_jsonb(NEW))
      WHERE value IS DISTINCT FROM (SELECT value FROM jsonb_each(to_jsonb(OLD)) WHERE key = jsonb_each.key)
    );
  ELSIF TG_OP = 'DELETE' THEN
    v_old_values := to_jsonb(OLD);
    v_new_values := NULL;
    v_changed_fields := ARRAY(SELECT jsonb_object_keys(to_jsonb(OLD)));
  END IF;
  
  -- Insert audit log
  INSERT INTO audit_logs (
    entity_type,
    entity_id,
    action,
    actor_id,
    actor_role,
    old_values,
    new_values,
    changed_fields,
    ip_address,
    user_agent
  ) VALUES (
    TG_TABLE_NAME,
    COALESCE(NEW.id, OLD.id),
    CASE TG_OP 
      WHEN 'INSERT' THEN 'create'
      WHEN 'UPDATE' THEN 'update'
      WHEN 'DELETE' THEN 'delete'
    END,
    v_actor_id,
    v_actor_role,
    v_old_values,
    v_new_values,
    v_changed_fields,
    inet_client_addr(),
    current_setting('request.jwt.claims', true)::json->>'user_agent'
  );
  
  RETURN COALESCE(NEW, OLD);
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

-- Attach triggers to core tables
CREATE TRIGGER audit_users
  AFTER INSERT OR UPDATE OR DELETE ON users
  FOR EACH ROW EXECUTE FUNCTION create_audit_log();

CREATE TRIGGER audit_resources
  AFTER INSERT OR UPDATE OR DELETE ON resources
  FOR EACH ROW EXECUTE FUNCTION create_audit_log();

CREATE TRIGGER audit_bookings
  AFTER INSERT OR UPDATE OR DELETE ON bookings
  FOR EACH ROW EXECUTE FUNCTION create_audit_log();

CREATE TRIGGER audit_payments
  AFTER INSERT OR UPDATE OR DELETE ON payments
  FOR EACH ROW EXECUTE FUNCTION create_audit_log();
```

---

## Module 17: Testing Structure (`tests`)

### 17.1 Test Organization
```
tests/
├── unit/
│   ├── utils/
│   │   ├── format.test.ts
│   │   ├── validation.test.ts
│   │   └── philippines.test.ts
│   ├── hooks/
│   │   ├── useDebounce.test.ts
│   │   └── useLocalStorage.test.ts
│   └── validators/
│       ├── auth.test.ts
│       ├── booking.test.ts
│       └── payment.test.ts
├── integration/
│   ├── api/
│   │   ├── bookings.test.ts
│   │   ├── payments.test.ts
│   │   └── users.test.ts
│   └── features/
│       ├── booking-flow.test.tsx
│       ├── payment-flow.test.tsx
│       └── auth-flow.test.tsx
├── e2e/
│   ├── auth.spec.ts
│   ├── bookings.spec.ts
│   ├── payments.spec.ts
│   ├── admin-panel.spec.ts
│   └── audit-logs.spec.ts
├── fixtures/
│   ├── users.json
│   ├── resources.json
│   ├── bookings.json
│   └── payments.json
├── setup/
│   ├── vitest.setup.ts
│   ├── msw.handlers.ts
│   └── test-utils.tsx
└── playwright.config.ts
```

---

## Module 18: CI/CD (`.github/workflows`)

### 18.1 Workflow Files
```
.github/workflows/
├── ci.yml              # Lint, typecheck, test on every PR
├── build.yml           # Build and deploy preview on merge to main
├── deploy.yml          # Production deployment
├── db-migrate.yml      # Database migrations on release
└── dependency-check.yml # Weekly dependency updates
```

### 18.2 CI Pipeline
```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm run lint

  typecheck:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm run typecheck

  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm run test:run

  build:
    runs-on: ubuntu-latest
    needs: [lint, typecheck, test]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm run build
```

---

## Summary: Module Dependencies

```
┌─────────────────────────────────────────────────────────────────┐
│                        APP LAYER                                │
├─────────────────────────────────────────────────────────────────┤
│  Auth → Bookings → Resources → Users → Payments → Audit        │
│    ↓         ↓           ↓         ↓         ↓         ↓       │
│  Dashboard ← Settings ← Shared Components ← Shared Hooks       │
├─────────────────────────────────────────────────────────────────┤
│                      SHARED LAYER                               │
│  Types → Validators → Constants → Utils → API → UI Components  │
├─────────────────────────────────────────────────────────────────┤
│                      DATA LAYER                                 │
│  Supabase/PostgreSQL → RLS → Triggers → Edge Functions         │
└─────────────────────────────────────────────────────────────────┘
```

---

## Next Steps for Implementation

1. **Start with Foundation** (Phase 1):
   - Set up Supabase project
   - Run migrations
   - Configure auth
   - Build layout components

2. **Build Core Features** (Phase 2-3):
   - Resources CRUD
   - Booking flow with availability
   - User management
   - Payment recording

3. **Add Intelligence** (Phase 4-5):
   - Audit logging (automatic via triggers)
   - Dashboard analytics
   - Export functionality

4. **Polish & Deploy** (Phase 6):
   - Error boundaries
   - Accessibility
   - CI/CD
   - Production deploy

---

*Document Version: 1.0*  
*Created: 2024*  
*For: Booking Management System - Philippines*