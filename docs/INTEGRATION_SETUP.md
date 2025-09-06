# Integration Setup Guide

## Introduction

This comprehensive guide walks you through setting up Clerk, Convex, and Vercel from scratch. Follow these steps in order to ensure all integrations work correctly together.

## Prerequisites

Before starting, ensure you have:
- Node.js 18+ installed
- A GitHub account (for Vercel deployment)
- Credit card for Clerk and Convex (both have generous free tiers)

## Phase 1: Project Setup

### 1. **Clone and Install Dependencies**

```bash
# Clone the starter template
git clone https://github.com/your-repo/elite-next-clerk-convex-starter.git
cd elite-next-clerk-convex-starter

# Install dependencies
pnpm install
# or npm install / yarn install / bun install
```

### 2. **Environment Configuration**

```bash
# Copy the environment template
cp .env.example .env.local
```

Open `.env.local` and keep it ready - we'll fill in the values as we configure each service.

## Phase 2: Convex Setup

### 1. **Create Convex Account**

1. Go to [https://dashboard.convex.dev](https://dashboard.convex.dev)
2. Sign up with GitHub or Google
3. Create a new project (e.g., "my-saas-app")

### 2. **Initialize Convex**

```bash
# Initialize Convex in your project
npx convex dev

# This will:
# - Create a convex/ directory (already exists in starter)
# - Set up your deployment
# - Start the development server
# - Generate API types
```

### 3. **Configure Convex Environment**

After running `npx convex dev`, you'll get:
- Deployment URL (e.g., `https://abc123.convex.cloud`)
- Deployment name (e.g., `abc123`)

Update your `.env.local`:
```bash
CONVEX_DEPLOYMENT=abc123
NEXT_PUBLIC_CONVEX_URL=https://abc123.convex.cloud
```

### 4. **Test Convex Connection**

```bash
# In another terminal, start Next.js
pnpm run dev

# Visit http://localhost:3000
# You should see the landing page without errors
```

## Phase 3: Clerk Authentication Setup

### 1. **Create Clerk Account**

1. Go to [https://dashboard.clerk.com](https://dashboard.clerk.com)
2. Sign up and create a new application
3. Choose "Next.js" as your framework

### 2. **Get Clerk Keys**

From your Clerk dashboard, copy:
- Publishable key (starts with `pk_test_`)
- Secret key (starts with `sk_test_`)

Update your `.env.local`:
```bash
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=pk_test_your_key_here
CLERK_SECRET_KEY=sk_test_your_key_here
```

### 3. **Configure JWT Template**

This is crucial for Clerk-Convex integration:

1. In Clerk Dashboard, go to **JWT Templates**
2. Click **New template**
3. Name it exactly: `convex`
4. Set the template as:

```json
{
  "iss": "{{iss}}",
  "sub": "{{user.id}}",
  "aud": ["convex"],
  "exp": "{{date.now + 3600}}",
  "iat": "{{date.now}}",
  "nbf": "{{date.now}}"
}
```

5. Save the template
6. Copy the **Issuer** URL (looks like `https://abc-123.clerk.accounts.dev`)

Update your `.env.local`:
```bash
NEXT_PUBLIC_CLERK_FRONTEND_API_URL=https://abc-123.clerk.accounts.dev
```

### 4. **Set Convex Environment Variables**

In your Convex dashboard:
1. Go to Settings → Environment Variables
2. Add:
   ```
   NEXT_PUBLIC_CLERK_FRONTEND_API_URL = https://abc-123.clerk.accounts.dev
   ```

### 5. **Configure Redirect URLs**

Update your `.env.local` with redirect settings:
```bash
NEXT_PUBLIC_CLERK_SIGN_IN_FORCE_REDIRECT_URL=/dashboard
NEXT_PUBLIC_CLERK_SIGN_UP_FORCE_REDIRECT_URL=/dashboard
NEXT_PUBLIC_CLERK_SIGN_IN_FALLBACK_REDIRECT_URL=/dashboard
NEXT_PUBLIC_CLERK_SIGN_UP_FALLBACK_REDIRECT_URL=/dashboard
```

### 6. **Test Authentication**

```bash
# Restart your development server
pnpm run dev

# Visit http://localhost:3000
# Click "Sign In" - you should see Clerk's auth interface
# Create a test account
# After signing up, you should be redirected to /dashboard
```

## Phase 4: Webhook Integration

### 1. **Expose Local Development**

For webhook testing, you need a public URL:

```bash
# Install ngrok if you haven't
npm install -g ngrok

# Expose your local server
ngrok http 3000

# Copy the HTTPS URL (e.g., https://abc123.ngrok.io)
```

### 2. **Configure Clerk Webhooks**

1. In Clerk Dashboard, go to **Webhooks**
2. Click **Add Endpoint**
3. Set URL to: `https://abc123.ngrok.io/clerk-users-webhook`
4. Select events:
   - `user.created`
   - `user.updated`
   - `user.deleted`
5. Click **Create**
6. Copy the **Signing Secret** (starts with `whsec_`)

### 3. **Add Webhook Secret to Convex**

In your Convex dashboard Environment Variables:
```
CLERK_WEBHOOK_SECRET = whsec_your_signing_secret_here
```

### 4. **Test Webhook Integration**

1. Keep `npx convex dev` running to see logs
2. In Clerk Dashboard, go to **Users**
3. Create or update a test user
4. Check Convex Dashboard logs - you should see webhook events being processed
5. Go to your database in Convex Dashboard - you should see user records

## Phase 5: Billing Setup (Optional)

### 1. **Enable Clerk Billing**

1. In Clerk Dashboard, go to **Billing**
2. Follow the setup wizard to configure:
   - Payment provider (Stripe recommended)
   - Tax settings
   - Business information

### 2. **Create Pricing Plans**

1. Create subscription plans (e.g., Free, Starter, Pro)
2. Set prices and features
3. Note the plan IDs/slugs

### 3. **Configure Payment Webhooks**

1. In Clerk Webhooks, add `paymentAttempt.updated` event
2. This will sync payment status to your Convex database

### 4. **Test Payment Flow**

1. Visit `/dashboard/payment-gated` while on a free plan
2. You should see the pricing table
3. Test the upgrade flow (use Stripe test cards)

## Phase 6: Vercel Deployment

### 1. **Create Vercel Account**

1. Go to [https://vercel.com](https://vercel.com)
2. Sign up with GitHub
3. Connect your repository

### 2. **Configure Production Environment**

In your Convex dashboard:
1. Create a new deployment for production
2. Run: `npx convex deploy --prod`
3. Copy the production URLs

### 3. **Set Vercel Environment Variables**

In Vercel Dashboard, add these environment variables:

```bash
# Convex
CONVEX_DEPLOYMENT=your_prod_deployment_name
NEXT_PUBLIC_CONVEX_URL=https://your-prod-deployment.convex.cloud

# Clerk  
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=pk_live_your_live_key
CLERK_SECRET_KEY=sk_live_your_live_key
NEXT_PUBLIC_CLERK_FRONTEND_API_URL=https://your-domain.clerk.accounts.dev

# Redirects
NEXT_PUBLIC_CLERK_SIGN_IN_FORCE_REDIRECT_URL=/dashboard
NEXT_PUBLIC_CLERK_SIGN_UP_FORCE_REDIRECT_URL=/dashboard
NEXT_PUBLIC_CLERK_SIGN_IN_FALLBACK_REDIRECT_URL=/dashboard
NEXT_PUBLIC_CLERK_SIGN_UP_FALLBACK_REDIRECT_URL=/dashboard
```

### 4. **Update Production Webhooks**

1. In Clerk Dashboard, update webhook URL to your production domain:
   `https://your-app.vercel.app/clerk-users-webhook`

### 5. **Deploy**

```bash
# Deploy to Vercel
vercel --prod

# Or push to main branch if auto-deployment is enabled
git push origin main
```

## Complete Environment File

Here's what your final `.env.local` should look like:

```bash
# Convex Configuration
CONVEX_DEPLOYMENT=your_development_deployment
NEXT_PUBLIC_CONVEX_URL=https://your-dev-deployment.convex.cloud

# Clerk Authentication & Billing
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=pk_test_your_publishable_key
CLERK_SECRET_KEY=sk_test_your_secret_key

# Clerk Frontend API URL (from JWT template)
NEXT_PUBLIC_CLERK_FRONTEND_API_URL=https://your-instance.clerk.accounts.dev

# Clerk Redirect URLs
NEXT_PUBLIC_CLERK_SIGN_IN_FORCE_REDIRECT_URL=/dashboard
NEXT_PUBLIC_CLERK_SIGN_UP_FORCE_REDIRECT_URL=/dashboard
NEXT_PUBLIC_CLERK_SIGN_IN_FALLBACK_REDIRECT_URL=/dashboard
NEXT_PUBLIC_CLERK_SIGN_UP_FALLBACK_REDIRECT_URL=/dashboard
```

## Troubleshooting Guide

### Common Issues and Solutions

#### 1. **"ConvexProviderWithClerk requires useAuth" Error**

**Cause**: Clerk provider not wrapping Convex provider correctly.

**Solution**: Ensure the provider order in `app/layout.tsx`:
```typescript
<ClerkProvider>
  <ConvexClientProvider>
    {children}
  </ConvexClientProvider>
</ClerkProvider>
```

#### 2. **"Invalid JWT" Errors**

**Cause**: JWT template misconfiguration.

**Solutions**:
- Verify JWT template name is exactly "convex"
- Check that issuer URL matches in both Clerk and Convex environments
- Ensure `NEXT_PUBLIC_CLERK_FRONTEND_API_URL` is set in both `.env.local` and Convex environment

#### 3. **Webhooks Not Working**

**Cause**: Various webhook configuration issues.

**Solutions**:
- Verify webhook URL is publicly accessible
- Check webhook secret is set in Convex environment (not `.env.local`)
- Ensure selected events include `user.created`, `user.updated`, `user.deleted`
- Check Convex function logs for error messages

#### 4. **Infinite Redirect Loops**

**Cause**: Middleware configuration or redirect URL issues.

**Solutions**:
- Check that protected routes in `middleware.ts` match your actual routes
- Verify redirect URLs don't cause circular redirects
- Ensure authentication pages are not protected by middleware

#### 5. **Database Connection Issues**

**Cause**: Environment variables or deployment configuration.

**Solutions**:
- Verify `NEXT_PUBLIC_CONVEX_URL` is correct
- Check that `npx convex dev` is running during development
- Ensure Convex deployment is active

#### 6. **Build/Deployment Failures**

**Cause**: Environment variables or build configuration.

**Solutions**:
- Run `pnpm run build` locally to catch build errors
- Verify all required environment variables are set in Vercel
- Check that Convex production deployment exists
- Ensure TypeScript errors are resolved

### Debugging Steps

#### 1. **Check Environment Variables**
```bash
# In your component or API route
console.log("Environment check:", {
  convexUrl: process.env.NEXT_PUBLIC_CONVEX_URL,
  clerkPublishable: process.env.NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY,
  // Never log secret keys!
});
```

#### 2. **Test Each Integration Separately**

**Test Convex**:
```typescript
// Create a simple test query
const testData = useQuery(api.users.current);
console.log("Convex data:", testData);
```

**Test Clerk**:
```typescript  
// Check authentication state
const { user, isLoaded } = useUser();
console.log("Clerk user:", { user, isLoaded });
```

#### 3. **Monitor Logs**

- **Convex**: Check function logs in Convex Dashboard
- **Vercel**: Check function logs in Vercel Dashboard  
- **Clerk**: Check webhook logs in Clerk Dashboard

### Getting Help

#### 1. **Official Documentation**
- [Convex Docs](https://docs.convex.dev)
- [Clerk Docs](https://clerk.com/docs)
- [Vercel Docs](https://vercel.com/docs)

#### 2. **Community Support**
- Convex Discord
- Clerk Discord
- Next.js GitHub Discussions

#### 3. **Debugging Checklist**

Before asking for help, verify:
- [ ] All environment variables are set correctly
- [ ] JWT template is configured with name "convex"
- [ ] Webhook endpoints are publicly accessible
- [ ] All services (Convex, Clerk) are properly configured
- [ ] Build passes locally with `pnpm run build`
- [ ] Console shows no obvious errors

## Advanced Configuration

### Custom Domain Setup

#### 1. **Vercel Custom Domain**
1. In Vercel Dashboard, go to Domains
2. Add your custom domain
3. Configure DNS as instructed

#### 2. **Update Clerk Configuration**
1. Add your custom domain to Clerk's allowed origins
2. Update redirect URLs to use custom domain
3. Update webhook URLs to use custom domain

### Multi-Environment Setup

#### 1. **Staging Environment**
```bash
# Create staging deployment in Convex
npx convex deploy --prod-url staging

# Create staging branch/deployment in Vercel
# Set staging-specific environment variables
```

#### 2. **Environment-Specific Configuration**
```typescript
// lib/config.ts
export const config = {
  isDevelopment: process.env.NODE_ENV === 'development',
  isProduction: process.env.NODE_ENV === 'production',
  convexUrl: process.env.NEXT_PUBLIC_CONVEX_URL!,
  clerkPublishableKey: process.env.NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY!,
};
```

### Performance Optimisation

#### 1. **Preload Critical Data**
```typescript
// app/dashboard/layout.tsx  
export default function DashboardLayout({ children }) {
  // Preload user data
  const user = useQuery(api.users.current);
  
  if (user === undefined) return <LoadingSpinner />;
  
  return <>{children}</>;
}
```

#### 2. **Implement Caching**
```typescript
// Use SWR patterns for non-real-time data
const { data: externalData } = useSWR('/api/external-data', fetcher);
```

## Next Steps

After completing the setup:

1. **Read the Architecture Docs**: Understand how the system works
   - [ARCHITECTURE_OVERVIEW.md](./ARCHITECTURE_OVERVIEW.md)
   
2. **Review Best Practices**: Learn security and performance patterns
   - [BEST_PRACTICES.md](./BEST_PRACTICES.md)
   
3. **Understand Each Provider**: Deep-dive into specific integrations
   - [CLERK_INTEGRATION.md](./CLERK_INTEGRATION.md)
   - [CONVEX_BACKEND.md](./CONVEX_BACKEND.md)
   - [VERCEL_DEPLOYMENT.md](./VERCEL_DEPLOYMENT.md)

4. **Customise for Your Needs**: Adapt the starter to your specific requirements
   - Add new database tables
   - Create custom Convex functions
   - Add new pages and features
   - Configure additional integrations

Your SaaS application is now ready for development!