# Booking Management System

A React-based booking management boilerplate with role-based access control, audit trails, and Philippines-specific payment handling.

## Tech Stack

- **Frontend**: React 18 + TypeScript + Vite
- **Styling**: Tailwind CSS + shadcn/ui (Radix UI)
- **State Management**: TanStack Query (server) + Zustand (client)
- **Forms**: React Hook Form + Zod
- **Authentication**: Supabase Auth (JWT + Row Level Security)
- **Database**: PostgreSQL (via Supabase)
- **Deployment**: Vercel (frontend) + Supabase (backend)

## Features

- **Role-Based Access Control**: Super Admin, Admin, User
- **Booking Management**: Multi-step creation, calendar view, availability checking
- **Resource Management**: Rooms, vehicles, equipment, spaces, services
- **Payment Recording**: Cash, GCash, PayMaya, Bank Transfer, Check
- **Audit Logs**: Automatic tracking of all changes
- **Dashboard**: Role-based analytics with charts
- **Philippines-Specific**: PHP currency, Asia/Manila timezone, local payment methods

## Getting Started

### Prerequisites

- Node.js 18+
- npm or pnpm
- Supabase account (for backend)

### Installation

```bash
# Clone the repository
git clone <your-repo-url>
cd booking-management

# Install dependencies
npm install

# Copy environment variables
cp .env.example .env

# Configure your Supabase credentials in .env
# VITE_SUPABASE_URL=your_supabase_url
# VITE_SUPABASE_ANON_KEY=your_anon_key

# Start development server
npm run dev
```

### Available Scripts

```bash
npm run dev          # Start development server
npm run build        # Build for production
npm run preview      # Preview production build
npm run lint         # Run ESLint
npm run typecheck    # Run TypeScript type checking
npm run test         # Run tests
npm run format       # Format code with Prettier
```

## Project Structure

```
booking-management/
├── src/
│   ├── app/                 # App-level config (providers, routes, layout)
│   ├── features/            # Feature modules
│   │   ├── auth/            # Authentication
│   │   ├── bookings/        # Booking management
│   │   ├── resources/       # Resource management
│   │   ├── users/           # User management
│   │   ├── payments/        # Payment recording
│   │   ├── audit/           # Audit logs
│   │   ├── dashboard/       # Dashboards
│   │   └── settings/        # System settings
│   ├── shared/              # Shared code
│   │   ├── components/      # Reusable UI components
│   │   ├── hooks/           # Custom hooks
│   │   ├── utils/           # Utility functions
│   │   ├── types/           # TypeScript types
│   │   ├── constants/       # App constants
│   │   ├── api/             # API client
│   │   └── validators/      # Zod schemas
│   └── styles/              # Global styles
├── supabase/                # Supabase config
│   ├── migrations/          # Database migrations
│   └── seed.sql             # Seed data
└── tests/                   # Tests
```

## Database Setup

1. Create a new Supabase project
2. Run migrations in order:
   ```bash
   # Using Supabase CLI
   supabase db push
   
   # Or manually run SQL files in supabase/migrations/
   ```
3. Configure Row Level Security policies
4. Set up storage buckets for file uploads

## Environment Variables

See `.env.example` for all required variables.

## Deployment

### Frontend (Vercel)
1. Connect your GitHub repository to Vercel
2. Add environment variables
3. Deploy

### Backend (Supabase)
- Database: Already hosted on Supabase
- Auth: Configured in Supabase dashboard
- Storage: Configure buckets in Supabase dashboard

## Documentation

- [Project Plan](PROJECT_PLAN.md) - High-level architecture
- [Module Details](MODULE_DETAILS.md) - Granular implementation details

## License

MIT