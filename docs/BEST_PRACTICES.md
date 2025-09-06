# Best Practices Guide

## Introduction

This guide outlines security, performance, and development workflow recommendations for building production-ready SaaS applications using Clerk, Convex, and Vercel. These practices are derived from real-world experience and platform-specific optimisations.

## Security Best Practices

### 1. **Environment Variable Management**

#### **Never Expose Secrets**
```bash
# ✅ Correct - Public values only
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=pk_test_...
NEXT_PUBLIC_CONVEX_URL=https://...

# ❌ Wrong - Secret in public variable
NEXT_PUBLIC_CLERK_SECRET_KEY=sk_test_... # Exposed to browser!
```

#### **Environment Separation**
```bash
# Development (.env.local)
CLERK_SECRET_KEY=sk_test_development_key

# Production (Vercel Dashboard)  
CLERK_SECRET_KEY=sk_live_production_key
```

#### **Convex Environment Variables**
```bash
# Set in Convex Dashboard, not .env.local
CLERK_WEBHOOK_SECRET=whsec_...
THIRD_PARTY_API_KEY=secret_key_...
```

**Why Separate**: Convex functions run independently and need their own environment configuration.

### 2. **Webhook Security**

#### **Always Validate Signatures**
```typescript
// convex/http.ts
async function validateRequest(req: Request): Promise<WebhookEvent | null> {
  const payloadString = await req.text();
  const svixHeaders = {
    "svix-id": req.headers.get("svix-id")!,
    "svix-timestamp": req.headers.get("svix-timestamp")!,
    "svix-signature": req.headers.get("svix-signature")!,
  };
  
  const wh = new Webhook(process.env.CLERK_WEBHOOK_SECRET!);
  try {
    return wh.verify(payloadString, svixHeaders) as unknown as WebhookEvent;
  } catch (error) {
    console.error("Invalid webhook signature", error);
    return null; // Reject invalid webhooks
  }
}
```

#### **Implement Idempotency**
```typescript
// Prevent duplicate processing
const existingPayment = await ctx.db
  .query("paymentAttempts")
  .withIndex("byPaymentId", (q) => q.eq("payment_id", data.payment_id))
  .unique();

if (existingPayment) {
  // Update existing record
  await ctx.db.patch(existingPayment._id, updateData);
} else {
  // Create new record
  await ctx.db.insert("paymentAttempts", newData);
}
```

### 3. **Authentication Security**

#### **Protect All Sensitive Routes**
```typescript
// middleware.ts - Comprehensive protection
const isProtectedRoute = createRouteMatcher([
  '/dashboard(.*)',
  '/api/user(.*)',
  '/settings(.*)',
  '/billing(.*)' // Don't forget billing pages!
])
```

#### **Server-Side Authentication Checks**
```typescript
// API routes - Always verify server-side
import { auth } from '@clerk/nextjs/server'

export async function GET() {
  const { userId } = await auth()
  
  if (!userId) {
    return new Response('Unauthorized', { status: 401 })
  }
  
  // Proceed with authenticated logic
}
```

#### **Convex Function Security**
```typescript
// Always get current user in functions that need auth
export const sensitiveData = query({
  handler: async (ctx) => {
    const user = await getCurrentUserOrThrow(ctx);
    
    // User is guaranteed to exist and be authenticated
    return await ctx.db
      .query("userSecrets")
      .withIndex("byUserId", (q) => q.eq("userId", user._id))
      .collect();
  },
});
```

### 4. **Data Validation**

#### **Validate All Inputs**
```typescript
// Use Convex validators for runtime safety
export const createTask = mutation({
  args: {
    title: v.string(),
    description: v.optional(v.string()),
    dueDate: v.optional(v.number()),
    priority: v.union(v.literal("low"), v.literal("medium"), v.literal("high"))
  },
  handler: async (ctx, args) => {
    // Arguments are guaranteed to match validation schema
    const user = await getCurrentUserOrThrow(ctx);
    
    // Additional business logic validation
    if (args.title.length < 3) {
      throw new ConvexError("Title must be at least 3 characters");
    }
    
    return await ctx.db.insert("tasks", {
      ...args,
      userId: user._id,
      createdAt: Date.now()
    });
  }
});
```

#### **Sanitise External Data**
```typescript
// Transform webhook data to prevent injection
export function transformWebhookData(data: any) {
  return {
    // Explicitly map fields - don't spread unknown data
    billing_date: Number(data.billing_date),
    charge_type: String(data.charge_type).substring(0, 50), // Limit length
    status: String(data.status).toLowerCase(),
    payer: {
      email: String(data.payer.email).toLowerCase().trim(),
      first_name: String(data.payer.first_name).substring(0, 100),
      user_id: String(data.payer.user_id).trim(),
    }
  };
}
```

## Performance Best Practices

### 1. **Database Query Optimisation**

#### **Always Use Indexes**
```typescript
// ✅ Efficient - Uses index
const user = await ctx.db
  .query("users")
  .withIndex("byExternalId", (q) => q.eq("externalId", clerkUserId))
  .unique();

// ❌ Inefficient - Full table scan
const user = await ctx.db
  .query("users")
  .filter((q) => q.eq(q.field("externalId"), clerkUserId))
  .first();
```

#### **Design Indexes for Query Patterns**
```typescript
// convex/schema.ts - Strategic index design
export default defineSchema({
  tasks: defineTable({
    title: v.string(),
    userId: v.id("users"),
    completed: v.boolean(),
    createdAt: v.number(),
    priority: v.string(),
  })
  // Index for user's tasks ordered by creation date
  .index("byUserCreated", ["userId", "createdAt"])
  // Index for user's incomplete tasks
  .index("byUserIncomplete", ["userId", "completed"])
  // Index for high priority tasks across users
  .index("byPriority", ["priority", "createdAt"])
});
```

#### **Pagination for Large Datasets**
```typescript
export const paginatedTasks = query({
  args: { 
    cursor: v.optional(v.string()),
    limit: v.number()
  },
  handler: async (ctx, { cursor, limit }) => {
    const user = await getCurrentUserOrThrow(ctx);
    
    return await ctx.db
      .query("tasks")
      .withIndex("byUserCreated", (q) => q.eq("userId", user._id))
      .order("desc") // Most recent first
      .paginate({ cursor, numItems: Math.min(limit, 100) }); // Limit max results
  },
});
```

### 2. **Frontend Performance**

#### **Optimistic Updates**
```typescript
// Update UI immediately, handle errors gracefully
const updateTask = useMutation(api.tasks.update);

const handleToggleComplete = async (taskId: string, completed: boolean) => {
  // UI updates immediately
  try {
    await updateTask({ taskId, completed });
    // Success - optimistic update becomes permanent
  } catch (error) {
    // Error - UI automatically reverts
    toast.error("Failed to update task");
  }
};
```

#### **Efficient Re-renders**
```typescript
// Use specific queries instead of fetching everything
const tasks = useQuery(api.tasks.incomplete); // ✅ Specific
const allData = useQuery(api.all.everything); // ❌ Over-fetching

// Memoize expensive computations
const sortedTasks = useMemo(() => {
  return tasks?.sort((a, b) => b.createdAt - a.createdAt) ?? [];
}, [tasks]);
```

#### **Loading States**
```typescript
function TaskList() {
  const tasks = useQuery(api.tasks.list);
  
  // Handle all states explicitly
  if (tasks === undefined) return <TaskSkeleton />; // Loading
  if (tasks.length === 0) return <EmptyState />; // No data
  
  return <TaskGrid tasks={tasks} />; // Data available
}
```

### 3. **Real-time Optimisation**

#### **Subscription Management**
```typescript
// Only subscribe to data you actually need
function UserDashboard({ userId }: { userId: string }) {
  // ✅ Subscribe only to current user's data
  const userData = useQuery(api.users.current);
  const userTasks = useQuery(api.tasks.byUser, { userId });
  
  // ❌ Don't subscribe to all users
  // const allUsers = useQuery(api.users.all);
}
```

#### **Batch Updates**
```typescript
// Batch related updates in a single mutation
export const createTaskWithSubtasks = mutation({
  args: {
    task: taskValidator,
    subtasks: v.array(subtaskValidator)
  },
  handler: async (ctx, { task, subtasks }) => {
    const taskId = await ctx.db.insert("tasks", task);
    
    // Batch insert subtasks
    await Promise.all(
      subtasks.map(subtask => 
        ctx.db.insert("subtasks", { ...subtask, taskId })
      )
    );
    
    return taskId;
  }
});
```

## Development Workflow Practices

### 1. **Environment Setup**

#### **Development Workflow**
```bash
# 1. Start Convex development server
npx convex dev

# 2. In another terminal, start Next.js
pnpm run dev

# 3. For webhook testing, use ngrok
ngrok http 3000
# Then update webhook URL in Clerk dashboard to ngrok URL + /clerk-users-webhook
```

#### **Code Organisation**
```
src/
├── app/
│   ├── (auth)/          # Authentication pages
│   ├── (landing)/       # Marketing pages  
│   ├── dashboard/       # App functionality
│   └── api/            # Vercel API routes (external integrations)
├── components/
│   ├── ui/             # shadcn/ui components
│   └── custom/         # App-specific components
├── convex/
│   ├── schema.ts       # Database schema
│   ├── users.ts        # User management
│   ├── [feature].ts    # Feature-specific functions
│   └── http.ts         # Webhook handlers
└── lib/
    └── utils.ts        # Shared utilities
```

### 2. **Testing Strategies**

#### **Convex Function Testing**
```typescript
// Test functions using Convex dashboard or HTTP client
import { ConvexHttpClient } from "convex/browser";

const convex = new ConvexHttpClient(process.env.CONVEX_URL!);

// Test queries
const result = await convex.query(api.users.current);
console.log("Query result:", result);

// Test mutations  
await convex.mutation(api.tasks.create, {
  title: "Test task",
  description: "Testing function"
});
```

#### **Webhook Testing**
```bash
# 1. Use ngrok to expose local development
ngrok http 3000

# 2. Update webhook URL in Clerk dashboard
https://abc123.ngrok.io/clerk-users-webhook

# 3. Trigger webhooks by creating/updating users in Clerk dashboard

# 4. Check Convex dashboard for function logs
```

#### **Integration Testing**
```typescript
// Test the full auth flow
describe('Authentication Flow', () => {
  it('redirects unauthenticated users', async () => {
    // Test middleware protection
    const response = await fetch('/dashboard');
    expect(response.status).toBe(302); // Redirect to sign-in
  });
  
  it('allows authenticated users', async () => {
    // Mock authentication
    // Test protected routes work correctly
  });
});
```

### 3. **Error Handling**

#### **Graceful Degradation**
```typescript
function Dashboard() {
  const user = useQuery(api.users.current);
  const tasks = useQuery(api.tasks.list);
  
  // Handle each query state independently
  return (
    <div>
      {user === undefined ? (
        <UserSkeleton />
      ) : user === null ? (
        <SignInPrompt />
      ) : (
        <UserHeader user={user} />
      )}
      
      {tasks === undefined ? (
        <TasksSkeleton />
      ) : tasks.length === 0 ? (
        <EmptyTasksState />
      ) : (
        <TasksList tasks={tasks} />
      )}
    </div>
  );
}
```

#### **Error Boundaries**
```typescript
// app/error.tsx - Global error boundary
'use client'

import { useEffect } from 'react'

export default function Error({
  error,
  reset,
}: {
  error: Error & { digest?: string }
  reset: () => void
}) {
  useEffect(() => {
    // Log error to monitoring service
    console.error('Application error:', error)
  }, [error])

  return (
    <div className="flex min-h-screen flex-col items-center justify-center">
      <h2 className="text-xl font-semibold">Something went wrong!</h2>
      <button
        onClick={reset}
        className="mt-4 px-4 py-2 bg-blue-500 text-white rounded"
      >
        Try again
      </button>
    </div>
  )
}
```

#### **Function Error Handling**
```typescript
export const risky = mutation({
  handler: async (ctx, args) => {
    try {
      const user = await getCurrentUserOrThrow(ctx);
      
      // Risky business logic
      const result = await processComplexOperation(user, args);
      
      return { success: true, result };
    } catch (error) {
      // Log error for debugging
      console.error("Complex operation failed:", {
        error: error.message,
        userId: ctx.auth.getUserIdentity()?.subject,
        args
      });
      
      // Return user-friendly error
      throw new ConvexError(
        error instanceof ConvexError 
          ? error.message 
          : "Operation failed. Please try again."
      );
    }
  }
});
```

### 4. **Monitoring and Debugging**

#### **Logging Best Practices**
```typescript
// Structured logging in Convex functions
export const debugMutation = mutation({
  handler: async (ctx, args) => {
    const user = await getCurrentUser(ctx);
    
    // Log with context
    console.log("Operation started", {
      operation: "debugMutation",
      userId: user?._id,
      timestamp: new Date().toISOString(),
      args
    });
    
    try {
      const result = await performOperation(args);
      
      console.log("Operation completed", {
        operation: "debugMutation",
        success: true,
        resultSize: JSON.stringify(result).length
      });
      
      return result;
    } catch (error) {
      console.error("Operation failed", {
        operation: "debugMutation",
        error: error.message,
        userId: user?._id,
        args
      });
      
      throw error;
    }
  }
});
```

#### **Performance Monitoring**
```typescript
// Track performance metrics
export const slowQuery = query({
  handler: async (ctx) => {
    const start = Date.now();
    
    const result = await expensiveOperation(ctx);
    
    const duration = Date.now() - start;
    if (duration > 1000) {
      console.warn("Slow query detected", {
        query: "slowQuery",
        duration: `${duration}ms`
      });
    }
    
    return result;
  }
});
```

## Deployment Best Practices

### 1. **Pre-deployment Checklist**

```bash
# 1. Run linter
pnpm run lint

# 2. Build locally to catch errors
pnpm run build

# 3. Deploy Convex functions first
npx convex deploy --prod

# 4. Update environment variables if needed
# 5. Deploy to Vercel
vercel --prod
```

### 2. **Environment Synchronisation**

```bash
# Keep environments in sync
# Development
NEXT_PUBLIC_CONVEX_URL=https://dev-deployment.convex.cloud

# Production  
NEXT_PUBLIC_CONVEX_URL=https://prod-deployment.convex.cloud

# Ensure webhook URLs match environment
# Dev: https://dev-domain.ngrok.io/clerk-users-webhook
# Prod: https://yourapp.com/clerk-users-webhook
```

### 3. **Rollback Strategy**

```bash
# If deployment fails, can rollback Convex independently
npx convex rollback --prod

# Vercel rollback via dashboard or CLI
vercel rollback
```

## Common Pitfalls and Solutions

### 1. **Authentication Issues**
- **Problem**: JWT template misconfiguration
- **Solution**: Verify template name is exactly "convex" and issuer URL is correct

### 2. **Webhook Failures**
- **Problem**: Webhook signature validation fails
- **Solution**: Ensure webhook secret is set in Convex environment (not .env.local)

### 3. **Performance Issues**
- **Problem**: Slow queries due to missing indexes
- **Solution**: Add indexes for commonly queried fields

### 4. **Real-time Updates Not Working**
- **Problem**: Data changes don't trigger re-renders
- **Solution**: Ensure you're using `useQuery` (not `fetch`) for reactive data

## Next Steps

- [ARCHITECTURE_OVERVIEW.md](./ARCHITECTURE_OVERVIEW.md) - Understand the system design
- [INTEGRATION_SETUP.md](./INTEGRATION_SETUP.md) - Step-by-step setup guide
- [CLERK_INTEGRATION.md](./CLERK_INTEGRATION.md) - Authentication best practices  
- [CONVEX_BACKEND.md](./CONVEX_BACKEND.md) - Backend development patterns