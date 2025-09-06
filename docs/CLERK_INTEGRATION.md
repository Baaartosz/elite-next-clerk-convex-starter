# Clerk Integration Guide

## Introduction

Clerk serves as the authentication and billing provider in this architecture. This guide explains how Clerk integrates with the rest of the system, why it's structured this way, and best practices for implementation.

## Why Clerk for SaaS Applications

### **Authentication Complexity Elimination**
- **Social Login Integration**: Pre-built connections to Google, GitHub, LinkedIn, etc.
- **Security Standards**: OWASP compliance, rate limiting, breach detection
- **User Management**: Profile management, organisation features, admin dashboards
- **Multi-factor Authentication**: Built-in 2FA/MFA capabilities

### **Billing Integration Benefits**
- **PCI Compliance**: Handles payment card security automatically
- **Subscription Management**: Recurring billing, plan changes, cancellations
- **Global Payments**: Multiple currencies and payment methods
- **Webhook Integration**: Real-time payment status updates

## Authentication Flow Deep Dive

### 1. **Initial Setup Architecture**

```typescript
// app/layout.tsx - Root provider setup
<ClerkProvider>
  <ConvexClientProvider>
    {children}
  </ConvexClientProvider>
</ClerkProvider>
```

**Why This Order**: Clerk must wrap Convex because Convex needs access to Clerk's authentication context.

### 2. **JWT Template Configuration**

Located in Clerk Dashboard → JWT Templates → Create "convex" template:

```json
{
  "iss": "https://your-clerk-frontend-api-url.clerk.accounts.dev",
  "sub": "{{user.id}}",
  "iat": "{{date.now}}",
  "exp": "{{date.now + 3600}}",
  "aud": ["convex"]
}
```

**System Impact**: This JWT structure allows Convex to:
- Identify users via the `sub` (subject) field
- Validate token authenticity via the `iss` (issuer)
- Enforce token expiration for security

### 3. **Middleware Protection Pattern**

```typescript
// middleware.ts
import { clerkMiddleware, createRouteMatcher } from '@clerk/nextjs/server'

const isProtectedRoute = createRouteMatcher(['/dashboard(.*)'])

export default clerkMiddleware(async (auth, req) => {
  if (isProtectedRoute(req)) await auth.protect()
})
```

**How It Works**:
1. Intercepts all requests before they reach page components
2. Checks if route matches protected pattern
3. Validates Clerk session token
4. Redirects to sign-in if unauthorized
5. Allows request to proceed if authenticated

### 4. **Convex Authentication Bridge**

```typescript
// components/ConvexClientProvider.tsx
const convex = new ConvexReactClient(process.env.NEXT_PUBLIC_CONVEX_URL)

return (
  <ConvexProviderWithClerk client={convex} useAuth={useAuth}>
    {children}
  </ConvexProviderWithClerk>
)
```

**Bridge Functionality**:
- `ConvexProviderWithClerk` automatically passes Clerk JWT tokens to Convex
- `useAuth` hook provides real-time authentication state
- Token refresh handled automatically

## Billing Integration Architecture

### 1. **Custom Pricing Component**

```typescript
// components/custom-clerk-pricing.tsx
<PricingTable
  appearance={{
    baseTheme: theme === "dark" ? dark : undefined,
    elements: {
      pricingTableCardTitle: { fontSize: 20 },
      pricingTableCardFee: { fontSize: 36, fontWeight: 800 }
    }
  }}
/>
```

**Why Custom Component**:
- **Theme Integration**: Matches your app's dark/light theme
- **Brand Consistency**: Custom styling to match design system
- **Responsive Design**: Mobile-optimised layouts
- **No Iframe**: Direct integration without embedded iframe limitations

### 2. **Payment Gating Pattern**

```typescript
// app/dashboard/payment-gated/page.tsx
<Protect
  condition={(has) => !has({ plan: "free_user" })}
  fallback={<UpgradeCard />}
>
  <FeaturesCard />
</Protect>
```

**How It Works**:
- `Protect` component checks user's plan status in real-time
- `condition` function evaluates subscription status
- `fallback` renders pricing table if unauthorized
- Content renders only for paying customers

## Webhook Integration System

### 1. **Webhook Endpoint Architecture**

```typescript
// convex/http.ts
http.route({
  path: "/clerk-users-webhook",
  method: "POST",
  handler: httpAction(async (ctx, request) => {
    const event = await validateRequest(request);
    
    switch (event.type) {
      case "user.created":
      case "user.updated":
        await ctx.runMutation(internal.users.upsertFromClerk, {
          data: event.data
        });
        break;
        
      case "paymentAttempt.updated":
        await ctx.runMutation(internal.paymentAttempts.savePaymentAttempt, {
          paymentAttemptData: transformWebhookData(event.data)
        });
        break;
    }
  })
});
```

**System Design**:
- **Validation**: Svix library validates webhook authenticity
- **Internal Functions**: Use `internal` functions for security
- **Event Routing**: Different handlers for different event types
- **Error Handling**: Returns appropriate HTTP status codes

### 2. **User Synchronisation Pattern**

```typescript
// convex/users.ts
export const upsertFromClerk = internalMutation({
  args: { data: v.any() as Validator<UserJSON> },
  async handler(ctx, { data }) {
    const userAttributes = {
      name: `${data.first_name} ${data.last_name}`,
      externalId: data.id, // Clerk user ID
    };

    const user = await userByExternalId(ctx, data.id);
    if (user === null) {
      await ctx.db.insert("users", userAttributes);
    } else {
      await ctx.db.patch(user._id, userAttributes);
    }
  }
});
```

**Why This Pattern**:
- **Upsert Logic**: Handles both new users and updates
- **External ID Mapping**: Links Clerk users to Convex users
- **Data Transformation**: Converts Clerk format to app format
- **Idempotency**: Safe to run multiple times

### 3. **Payment Webhook Handling**

```typescript
// convex/paymentAttemptTypes.ts - Complex validation structure
export const paymentAttemptValidators = {
  billing_date: v.number(),
  charge_type: v.string(),
  status: v.string(),
  payer: v.object({
    email: v.string(),
    user_id: v.string(),
  }),
  subscription_items: v.array(v.object({
    plan: v.object({
      name: v.string(),
      slug: v.string(),
      amount: v.number(),
    })
  }))
};
```

**Why Complex Validation**:
- **Type Safety**: Ensures webhook data matches expected structure
- **Runtime Validation**: Catches malformed webhook data
- **Documentation**: Validation serves as schema documentation
- **Error Prevention**: Prevents database corruption from bad data

## Environment Configuration

### **Required Environment Variables**

```bash
# Authentication
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=pk_test_...
CLERK_SECRET_KEY=sk_test_...
NEXT_PUBLIC_CLERK_FRONTEND_API_URL=https://...clerk.accounts.dev

# Redirects (UX optimisation)
NEXT_PUBLIC_CLERK_SIGN_IN_FORCE_REDIRECT_URL=/dashboard
NEXT_PUBLIC_CLERK_SIGN_UP_FORCE_REDIRECT_URL=/dashboard
```

**Configuration Impact**:
- **Publishable Key**: Client-side authentication
- **Secret Key**: Server-side API calls
- **Frontend API URL**: JWT token validation
- **Redirect URLs**: Post-authentication user experience

### **Webhook Security**

```bash
# In Convex Dashboard (not .env.local)
CLERK_WEBHOOK_SECRET=whsec_...
```

**Why Convex Environment**: Webhook secret must be accessible to Convex functions but not exposed to client-side code.

## Best Practices

### **1. Error Handling**
```typescript
// Graceful degradation example
const { user } = useUser();
if (!user) {
  return <LoadingSpinner />;
}

// User is guaranteed to exist here
```

### **2. Type Safety**
```typescript
// Use Clerk's types
import { UserJSON } from "@clerk/backend";

// Validate webhook data
args: { data: v.any() as Validator<UserJSON> }
```

### **3. Security**
```typescript
// Always validate webhook signatures
const wh = new Webhook(process.env.CLERK_WEBHOOK_SECRET!);
return wh.verify(payloadString, svixHeaders);
```

### **4. User Experience**
- Use loading states during authentication checks
- Provide clear error messages for failed payments
- Implement proper redirect flows after authentication

## Common Integration Patterns

### **Protected API Routes**
```typescript
// For server-side data fetching
import { auth } from '@clerk/nextjs/server'

export async function GET() {
  const { userId } = await auth()
  if (!userId) {
    return new Response('Unauthorized', { status: 401 })
  }
  // Proceed with authenticated logic
}
```

### **Conditional UI Rendering**
```typescript
import { useUser } from '@clerk/nextjs'

export function Dashboard() {
  const { user, isLoaded, isSignedIn } = useUser()
  
  if (!isLoaded) return <Loading />
  if (!isSignedIn) return <SignInPrompt />
  
  return <DashboardContent user={user} />
}
```

## Troubleshooting Guide

### **JWT Template Issues**
- Verify the template name is exactly "convex"
- Ensure the Issuer URL matches your Clerk instance
- Check that the audience includes "convex"

### **Webhook Failures**
- Verify webhook endpoint URL is publicly accessible
- Check that webhook secret matches in both Clerk and Convex
- Ensure webhook events are enabled for required event types

### **Authentication Loops**
- Check that redirect URLs match your domain
- Verify middleware configuration covers all protected routes
- Ensure JWT template is properly configured

## Next Steps

- [CONVEX_BACKEND.md](./CONVEX_BACKEND.md) - Backend integration patterns
- [VERCEL_DEPLOYMENT.md](./VERCEL_DEPLOYMENT.md) - Deployment configuration
- [INTEGRATION_SETUP.md](./INTEGRATION_SETUP.md) - Step-by-step setup guide