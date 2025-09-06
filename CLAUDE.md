# 

## Commands

### Development
- `pnpm run dev` - Start development server with Turbopack
- `pnpm run build` - Build for production
- `pnpm start` - Start production server
- `pnpm run lint` - Run ESLint
- `npx convex dev` - Run Convex development server

### Convex Setup
- Initialize Convex: `npx convex dev`
- Convex functions are in the `convex/` directory

## Architecture Overview

This is a modern SaaS starter built with Next.js 15, Convex (real-time database), Clerk (authentication & billing), and TailwindCSS v4. The application follows a clean separation between frontend (Next.js app), backend (Convex functions), and authentication/payments (Clerk services).

### Key Architecture Patterns

**Frontend Structure:**
- Next.js 15 App Router with route groups
- `app/(landing)/` - Public marketing pages
- `app/dashboard/` - Protected user dashboard with sidebar layout
- `app/not-found.tsx` - Custom 404 page with animations

**Backend (Convex):**
- Real-time database with TypeScript-first schema in `convex/schema.ts`
- Public functions: queries/mutations accessible from frontend
- Internal functions: server-side only functions prefixed with `internal`
- Webhook handlers in `convex/http.ts` for Clerk integration
- New function syntax with explicit args/returns validators

**Authentication & Payments:**
- Clerk handles all user management, authentication, and billing
- Protected routes via middleware in `middleware.ts`
- User sync between Clerk and Convex via webhooks
- Payment tracking in Convex `paymentAttempts` table

### Database Schema

**Users Table:**
- `name: string` - User display name
- `externalId: string` - Clerk user ID for sync
- Index: `byExternalId` for efficient Clerk lookups

**PaymentAttempts Table:**
- Tracks Clerk billing webhooks
- Indexes: `byPaymentId`, `byUserId`, `byPayerUserId`

### Component Architecture

**UI System:**
- shadcn/ui components in `components/ui/`
- Custom animation components from React Bits, Motion Primitives
- Theme system with dark/light mode support
- TailwindCSS v4 with custom design tokens

**Layout Patterns:**
- Landing page uses grouped layout `app/(landing)/`
- Dashboard uses sidebar layout with `app/dashboard/layout.tsx`
- Responsive design with mobile-first approach

### Convex Function Patterns

**Always use new function syntax:**
```typescript
export const functionName = query({
  args: { param: v.string() },
  returns: v.array(v.object({ ... })),
  handler: async (ctx, args) => { ... }
});
```

**Internal vs Public Functions:**
- Use `internal` functions for server-side logic
- Use `query/mutation` for client-accessible functions
- Call internal functions via `ctx.runQuery(internal.module.function, args)`

**Database Operations:**
- Use indexes defined in schema for efficient queries
- Follow `.withIndex("indexName", q => q.eq("field", value))` pattern
- Use `.order("desc")` for reverse chronological ordering

### Authentication Flow

1. Clerk handles sign-in/sign-up UI and logic
2. Users redirected to `/dashboard` after authentication
3. Middleware protects `/dashboard/*` routes
4. Webhook syncs user data to Convex on user events
5. Frontend queries Convex for user-specific data

### Payment Integration

1. Custom Clerk pricing component renders subscription plans
2. Clerk handles payment processing and subscription management
3. `paymentAttempt.updated` webhooks sync payment status to Convex
4. Payment-gated content checks subscription status
5. Real-time updates via Convex subscriptions

### Environment Configuration

Required environment variables are documented in README.md. Key variables:
- Convex deployment URLs
- Clerk authentication keys  
- Webhook secrets for secure communication
- JWT template configuration for Clerk-Convex integration

### Styling and Theming

- TailwindCSS v4 with CSS-in-JS support
- Global styles in `app/globals.css`
- Component-level styling follows shadcn/ui patterns
- Theme customisation via CSS custom properties
- Motion/animation libraries: Framer Motion, React Bits

### Development Workflow

1. Run `npx convex dev` for backend development
2. Run `pnpm run dev` for frontend development with Turbopack
3. Use `pnpm run lint` before commits
4. Test webhook integration with ngrok or similar tools
5. Deploy to Vercel with environment variables configured