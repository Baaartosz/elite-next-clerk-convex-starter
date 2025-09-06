# Vercel Deployment & Server-Side Architecture

## Introduction

This guide explains how Vercel fits into the SaaS architecture, what server-side code can be added to the deployment, and how Convex changes the traditional backend approach. Understanding these patterns is crucial for making informed decisions about where to implement different types of functionality.

## Vercel's Role in the Architecture

### **Primary Responsibilities**
- **Static Site Hosting**: Serves pre-built HTML, CSS, and JavaScript
- **Edge Function Runtime**: Executes server-side logic at edge locations
- **Build Optimisation**: Turbopack integration for fast builds
- **Environment Management**: Secure environment variable handling
- **CDN & Caching**: Global content distribution with intelligent caching

### **Integration Points**
```
Frontend (Next.js) → Vercel Edge Runtime → Convex Functions
                   ↓
           Clerk Authentication
```

## Server-Side Possibilities on Vercel

### 1. **API Routes in Next.js**

You can add traditional API routes to handle server-side logic:

```typescript
// app/api/custom-endpoint/route.ts
import { auth } from '@clerk/nextjs/server'
import { NextRequest, NextResponse } from 'next/server'

export async function GET(request: NextRequest) {
  // Server-side authentication
  const { userId } = await auth()
  
  if (!userId) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
  }

  // Custom server-side logic
  const customData = await processCustomLogic(userId)
  
  return NextResponse.json({ data: customData })
}

async function processCustomLogic(userId: string) {
  // This runs on Vercel's servers, not Convex
  // Suitable for:
  // - Third-party API integrations  
  // - File processing
  // - Custom authentication flows
  // - Webhook handling that doesn't need real-time updates
  
  return { processed: true, userId }
}
```

**When to Use API Routes**:
- Third-party service integrations (payment processors, email services)
- File upload/download handling
- Custom webhook endpoints (non-Convex related)
- Server-side rendering data fetching
- Edge-specific logic (geolocation, A/B testing)

### 2. **Edge Functions**

```typescript
// middleware.ts - Already implemented
export default clerkMiddleware(async (auth, req) => {
  // This runs on Vercel's edge network
  if (isProtectedRoute(req)) await auth.protect()
})

// Custom edge function example
export async function middleware(request: NextRequest) {
  // Custom edge logic
  const country = request.geo?.country
  const response = NextResponse.next()
  
  // Add custom headers based on location
  response.headers.set('x-user-country', country || 'unknown')
  
  return response
}
```

**Edge Function Benefits**:
- **Low Latency**: Execute close to users globally
- **No Cold Starts**: Always ready to execute
- **Request Modification**: Modify requests/responses in flight

### 3. **Server Actions** 

```typescript
// app/dashboard/actions.ts
'use server'

import { auth } from '@clerk/nextjs/server'
import { redirect } from 'next/navigation'

export async function handleFormSubmission(formData: FormData) {
  const { userId } = await auth()
  
  if (!userId) {
    redirect('/sign-in')
  }

  // Process form data server-side
  const result = await processFormData(formData, userId)
  
  // Can call Convex functions from here if needed
  // const convexClient = new ConvexHttpClient(process.env.CONVEX_URL!)
  // await convexClient.mutation(api.custom.updateData, { result })
  
  return { success: true, result }
}
```

**Server Actions Use Cases**:
- Form processing with validation
- File uploads to cloud storage
- Email sending
- Data transformation before sending to Convex

### 4. **Server Components**

```typescript
// app/dashboard/analytics/page.tsx
import { auth } from '@clerk/nextjs/server'
import { ConvexHttpClient } from 'convex/browser'
import { api } from '@/convex/_generated/api'

export default async function AnalyticsPage() {
  const { userId } = await auth()
  
  // Server-side data fetching
  const convex = new ConvexHttpClient(process.env.CONVEX_URL!)
  const userData = await convex.query(api.users.current)
  
  // This component renders on the server
  return (
    <div>
      <h1>Analytics for {userData?.name}</h1>
      <AnalyticsChart initialData={userData} />
    </div>
  )
}
```

**Server Component Benefits**:
- **SEO**: Pre-rendered content
- **Performance**: Reduced client-side JavaScript
- **Security**: Sensitive logic runs server-side

## Convex vs Traditional Backend

### **Traditional Backend Pattern**
```
Frontend → API Server → Database
         ↓
    Manual state sync
    WebSocket management
    Cache invalidation
    Authentication middleware
```

### **Convex + Vercel Pattern**
```
Frontend → Convex Functions → Database (built-in)
         ↓
    Automatic real-time sync
    Built-in authentication
    Optimistic updates
    Type-safe client generation
```

### **Hybrid Approach** (This Starter Kit)
```
Frontend → Vercel API Routes (for external integrations)
        ↓
        → Convex Functions (for core app logic)
        ↓  
        → Convex Database (for app data)
```

## What Should Go Where?

### **Put in Convex Functions**
✅ **Core business logic**
✅ **User data management**  
✅ **Real-time features**
✅ **Database operations**
✅ **Authentication-dependent logic**
✅ **Cross-user data sharing**

```typescript
// convex/tasks.ts - Perfect for Convex
export const createTask = mutation({
  args: { title: v.string(), description: v.string() },
  handler: async (ctx, { title, description }) => {
    const user = await getCurrentUserOrThrow(ctx)
    
    return await ctx.db.insert("tasks", {
      title,
      description,
      userId: user._id,
      completed: false,
      createdAt: Date.now()
    })
  }
})
```

### **Put in Vercel API Routes**
✅ **Third-party integrations**
✅ **File processing**
✅ **Email sending**
✅ **Webhook handling (external services)**
✅ **Custom authentication flows**
✅ **Data imports/exports**

```typescript
// app/api/send-email/route.ts - Perfect for API Routes
export async function POST(request: NextRequest) {
  const { userId } = await auth()
  const { to, subject, body } = await request.json()
  
  // Send email using external service
  await emailService.send({
    to,
    subject,
    body,
    from: 'noreply@yourapp.com'
  })
  
  return NextResponse.json({ sent: true })
}
```

### **Put in Edge Functions/Middleware**
✅ **Request routing**
✅ **Authentication checks**
✅ **Rate limiting**
✅ **A/B testing**  
✅ **Geolocation logic**
✅ **Security headers**

## Environment Configuration

### **Vercel Environment Variables**
```bash
# .env.local (development)
CONVEX_DEPLOYMENT=your_deployment
NEXT_PUBLIC_CONVEX_URL=https://your-url.convex.cloud
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=pk_test_...
CLERK_SECRET_KEY=sk_test_...

# Additional Vercel-specific variables
VERCEL_URL=auto-populated-by-vercel
EMAIL_API_KEY=for_api_routes_email_sending
STORAGE_API_KEY=for_file_upload_handling
```

### **Production Deployment**
```bash
# Set in Vercel Dashboard
vercel env add CONVEX_DEPLOYMENT production
vercel env add CLERK_SECRET_KEY production
vercel env add EMAIL_API_KEY production
```

### **Environment Variable Access**
```typescript
// Server-side (API routes, server components)
const secretKey = process.env.CLERK_SECRET_KEY // ✅ Available

// Client-side (components, hooks)
const publicKey = process.env.NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY // ✅ Available
const secretKey = process.env.CLERK_SECRET_KEY // ❌ undefined
```

## Deployment Architecture

### **Build Process**
1. Vercel receives code push
2. Runs `next build` with Turbopack
3. Generates static assets and server functions
4. Deploys to global edge network
5. Updates environment variables
6. Connects to Convex deployment

### **Function Deployment**
```typescript
// These become Vercel serverless functions
app/api/*/route.ts → Vercel Functions
middleware.ts → Edge Function
app/**/page.tsx → Static pages or server functions (based on usage)
```

### **Convex Deployment**
```bash
# Convex functions deploy separately
npx convex deploy --prod

# Vercel deployment references Convex via environment variables
NEXT_PUBLIC_CONVEX_URL=https://your-prod-deployment.convex.cloud
```

## Performance Considerations

### **Vercel Optimisations**
- **Edge Caching**: Static assets cached globally
- **Server Function Caching**: Intelligent caching of dynamic content  
- **Image Optimisation**: Automatic image resizing and format conversion
- **Bundle Analysis**: Automatic bundle size optimisation

### **Convex Optimisations**  
- **Real-time Sync**: Only sends changed data, not full re-renders
- **Query Caching**: Automatic query result caching
- **Connection Pooling**: Efficient database connection management

## Scaling Patterns

### **Horizontal Scaling**
- **Vercel**: Automatically scales based on traffic
- **Convex**: Automatically scales functions and database
- **Combined**: Near-infinite scaling capability

### **Regional Distribution**
```typescript
// Vercel edge functions run globally
export const config = {
  runtime: 'edge',
  regions: ['iad1', 'sfo1', 'fra1'] // Optional: specify regions
}
```

### **Database Scaling**
- **Convex**: Handles scaling automatically
- **No connection limits**: Unlike traditional databases
- **Global distribution**: Data replicated globally for performance

## Monitoring and Debugging

### **Vercel Analytics**
```typescript
// Built-in analytics
import { Analytics } from '@vercel/analytics/react'

export default function RootLayout({ children }) {
  return (
    <html>
      <body>
        {children}
        <Analytics />
      </body>
    </html>
  )
}
```

### **Convex Debugging**
```typescript
// Built-in logging in Convex dashboard
export const debugFunction = mutation({
  handler: async (ctx, args) => {
    console.log("Debug info:", args) // Visible in Convex dashboard
    // Function logic
  }
})
```

### **Error Handling**
```typescript
// Vercel API route error handling
export async function POST(request: NextRequest) {
  try {
    // Logic here
  } catch (error) {
    console.error("API route error:", error)
    return NextResponse.json(
      { error: "Internal server error" },
      { status: 500 }
    )
  }
}
```

## Security Considerations

### **API Route Security**
```typescript
import { rateLimit } from '@/lib/rate-limit'

export async function POST(request: NextRequest) {
  // Rate limiting
  const identifier = request.ip ?? '127.0.0.1'
  const result = await rateLimit.limit(identifier)
  
  if (!result.success) {
    return NextResponse.json(
      { error: 'Too many requests' },
      { status: 429 }
    )
  }
  
  // Authentication
  const { userId } = await auth()
  if (!userId) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
  }
  
  // Process request
}
```

### **Environment Variable Security**
- Never expose secrets to client-side code
- Use `NEXT_PUBLIC_` prefix only for truly public values
- Store sensitive keys in Vercel dashboard, not in code

## Migration Strategies

### **From Traditional Backend**
1. **Phase 1**: Move authentication to Clerk
2. **Phase 2**: Move real-time features to Convex  
3. **Phase 3**: Gradually move business logic to Convex
4. **Phase 4**: Keep integration APIs on Vercel

### **Adding Server-Side Logic**
1. Start with API routes for new features
2. Evaluate if feature needs real-time capabilities
3. If yes, move to Convex functions
4. If no, keep in API routes

## Best Practices

### **Code Organisation**
```
app/
├── api/                    # Vercel API routes
│   ├── webhooks/          # External service webhooks  
│   ├── integrations/      # Third-party API calls
│   └── uploads/           # File handling
├── dashboard/             # App pages (can use server components)
└── (auth)/               # Authentication pages

convex/                    # Convex functions
├── users.ts              # User management
├── tasks.ts              # Core business logic
└── http.ts               # Convex webhook handlers
```

### **Decision Framework**
Ask these questions when deciding where to put logic:

1. **Does it need real-time updates?** → Convex
2. **Does it integrate with external APIs?** → Vercel API routes  
3. **Is it authentication/routing logic?** → Vercel middleware
4. **Does it need to run at the edge globally?** → Edge functions
5. **Is it core business logic with database operations?** → Convex

## Next Steps

- [ARCHITECTURE_OVERVIEW.md](./ARCHITECTURE_OVERVIEW.md) - System overview
- [BEST_PRACTICES.md](./BEST_PRACTICES.md) - Security and performance  
- [INTEGRATION_SETUP.md](./INTEGRATION_SETUP.md) - Setup procedures