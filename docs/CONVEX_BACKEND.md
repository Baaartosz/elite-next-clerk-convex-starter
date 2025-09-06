# Convex Backend Architecture Guide

## Introduction

Convex serves as the real-time backend for this SaaS application. This guide explains why Convex functions are structured the way they are, how the real-time capabilities work at a system level, and best practices for backend development.

## Why Convex for Modern SaaS

### **Real-time Without Complexity**
- **Automatic WebSocket Management**: No need to manage connections, reconnections, or message queuing
- **Optimistic Updates**: UI updates immediately, with automatic rollback on conflicts
- **Live Queries**: Database queries automatically re-run when underlying data changes
- **Offline Support**: Built-in offline capabilities with automatic sync when online

### **Type Safety Across the Stack**
- **Generated Client**: TypeScript types automatically generated from backend functions
- **Runtime Validation**: Input/output validation enforced at runtime
- **End-to-End Types**: Frontend knows exactly what backend functions expect and return

### **Serverless with Persistent State**
- **No Cold Starts**: Functions optimised for instant execution
- **Persistent Connections**: Real-time subscriptions maintained across function executions
- **Automatic Scaling**: Scales from zero to thousands of concurrent users automatically

## Function Architecture Deep Dive

### 1. **Why Explicit Validators**

```typescript
// convex/users.ts - New function syntax
export const current = query({
  args: {},
  returns: v.union(
    v.null(),
    v.object({
      _id: v.id("users"),
      name: v.string(),
      externalId: v.string(),
    })
  ),
  handler: async (ctx) => {
    return await getCurrentUser(ctx);
  },
});
```

**System-Level Reasons**:

1. **Runtime Type Safety**: Validates data at the boundary between client and server
2. **Client Generation**: Enables automatic TypeScript client generation
3. **Documentation**: Serves as executable documentation of API contracts
4. **Version Safety**: Prevents breaking changes by enforcing consistent interfaces
5. **Performance**: Optimises serialisation/deserialisation of data over the wire

### 2. **Internal vs Public Function Pattern**

```typescript
// Public function (client-accessible)
export const current = query({
  args: {},
  returns: v.union(v.null(), userValidator),
  handler: async (ctx) => {
    return await getCurrentUser(ctx);
  },
});

// Internal function (server-only)
export const upsertFromClerk = internalMutation({
  args: { data: v.any() as Validator<UserJSON> },
  handler: async (ctx, { data }) => {
    // Implementation
  },
});
```

**Architectural Benefits**:

- **Security Boundary**: Internal functions can't be called from frontend
- **Composition**: Internal functions can be called from other backend functions
- **Webhooks**: Webhook handlers can securely call internal functions
- **Business Logic**: Complex operations can be broken into internal components

### 3. **Authentication Integration**

```typescript
export async function getCurrentUser(ctx: QueryCtx) {
  const identity = await ctx.auth.getUserIdentity();
  if (identity === null) {
    return null;
  }
  return await userByExternalId(ctx, identity.subject);
}
```

**How It Works**:
1. `ctx.auth.getUserIdentity()` validates the JWT token from Clerk
2. Extracts user ID from the `subject` field
3. Queries database using the external ID (Clerk user ID)
4. Returns user record or null if not found

## Database Schema and Indexing Strategy

### 1. **Schema Definition**

```typescript
// convex/schema.ts
export default defineSchema({
  users: defineTable({
    name: v.string(),
    externalId: v.string(), // Clerk user ID
  }).index("byExternalId", ["externalId"]),
  
  paymentAttempts: defineTable(paymentAttemptSchemaValidator)
    .index("byPaymentId", ["payment_id"])
    .index("byUserId", ["userId"])
    .index("byPayerUserId", ["payer.user_id"]),
});
```

**Index Strategy**:
- **Primary Lookups**: Index fields used for finding specific records
- **Foreign Keys**: Index relationship fields for efficient joins
- **Webhook Queries**: Index fields used in webhook processing
- **User Queries**: Index fields commonly filtered by users

### 2. **Query Optimisation Patterns**

```typescript
// Efficient query using index
async function userByExternalId(ctx: QueryCtx, externalId: string) {
  return await ctx.db
    .query("users")
    .withIndex("byExternalId", (q) => q.eq("externalId", externalId))
    .unique(); // .unique() is more efficient than .first() when you expect one result
}
```

**Performance Considerations**:
- **Index Selection**: Always use indexes for queries when possible
- **Query Specificity**: More specific queries perform better
- **Unique vs First**: Use `.unique()` when expecting exactly one result

## Real-time Synchronisation Mechanics

### 1. **Live Query System**

```typescript
// Frontend component
function UserProfile() {
  const user = useQuery(api.users.current);
  
  if (user === undefined) return <Loading />; // Query in progress
  if (user === null) return <NotSignedIn />; // No user found
  
  return <ProfileDisplay user={user} />; // User data available
}
```

**How Real-time Works**:
1. Component subscribes to query result
2. Convex maintains WebSocket connection
3. When user data changes (via webhook or mutation), query re-executes
4. Component automatically re-renders with new data
5. No manual refresh or state management needed

### 2. **Optimistic Updates Pattern**

```typescript
// Frontend mutation with optimistic update
const updateProfile = useMutation(api.users.update);

const handleUpdate = async (newData) => {
  // UI updates immediately (optimistic)
  await updateProfile(newData);
  // If mutation fails, UI automatically reverts
};
```

**System Behaviour**:
- UI updates immediately when mutation is called
- Mutation executes on server
- If successful, optimistic update becomes permanent
- If failed, UI automatically reverts to previous state

### 3. **Cross-User Real-time Updates**

```typescript
// When user A changes data that user B is viewing
// User B's interface updates automatically without any additional code
```

**Implementation**: Convex tracks all active subscriptions and automatically pushes updates to relevant clients.

## HTTP Route Handling

### 1. **Webhook Architecture**

```typescript
// convex/http.ts
const http = httpRouter();

http.route({
  path: "/clerk-users-webhook",
  method: "POST",
  handler: httpAction(async (ctx, request) => {
    const event = await validateRequest(request);
    
    switch (event.type) {
      case "user.created":
        await ctx.runMutation(internal.users.upsertFromClerk, {
          data: event.data
        });
        break;
    }
    
    return new Response(null, { status: 200 });
  }),
});
```

**System Design**:
- **HTTP Actions**: Handle external HTTP requests (webhooks, API calls)
- **Internal Mutations**: Process webhook data securely
- **Error Handling**: Return appropriate HTTP status codes
- **Validation**: Verify webhook authenticity before processing

### 2. **Request Validation Pattern**

```typescript
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
    console.error("Error verifying webhook event", error);
    return null;
  }
}
```

**Security Benefits**:
- **Signature Verification**: Ensures webhooks come from expected source
- **Replay Protection**: Prevents replay attacks via timestamp validation
- **Error Handling**: Gracefully handles malformed requests

## Data Validation and Type Safety

### 1. **Complex Validator Structures**

```typescript
// convex/paymentAttemptTypes.ts
export const paymentAttemptValidators = {
  billing_date: v.number(),
  payer: v.object({
    email: v.string(),
    first_name: v.string(),
    last_name: v.string(),
    user_id: v.string(),
  }),
  subscription_items: v.array(v.object({
    plan: v.object({
      name: v.string(),
      slug: v.string(),
      amount: v.number(),
    })
  })),
  totals: v.object({
    grand_total: v.object({
      amount: v.number(),
      amount_formatted: v.string(),
      currency: v.string(),
    })
  })
};
```

**Why Complex Validation**:
- **Webhook Data Safety**: Ensures external data matches expected structure
- **Runtime Type Checking**: Catches schema mismatches at runtime
- **Documentation**: Validators serve as executable documentation
- **Client Generation**: Enables accurate TypeScript type generation

### 2. **Data Transformation Pattern**

```typescript
export function transformWebhookData(data: any) {
  return {
    billing_date: data.billing_date,
    charge_type: data.charge_type,
    status: data.status,
    payer: {
      email: data.payer.email,
      first_name: data.payer.first_name,
      last_name: data.payer.last_name,
      user_id: data.payer.user_id,
    },
    // ... more transformations
  };
}
```

**Purpose**:
- **Data Normalisation**: Converts external format to internal format
- **Security**: Filters out unwanted fields from external data
- **Consistency**: Ensures all data follows internal conventions

## Function Composition Patterns

### 1. **Reusable Query Components**

```typescript
// Shared query logic
async function userByExternalId(ctx: QueryCtx, externalId: string) {
  return await ctx.db
    .query("users")
    .withIndex("byExternalId", (q) => q.eq("externalId", externalId))
    .unique();
}

// Used in multiple functions
export const current = query({
  handler: async (ctx) => {
    const identity = await ctx.auth.getUserIdentity();
    if (!identity) return null;
    return await userByExternalId(ctx, identity.subject);
  },
});

export const upsertFromClerk = internalMutation({
  handler: async (ctx, { data }) => {
    const existingUser = await userByExternalId(ctx, data.id);
    // ... upsert logic
  },
});
```

**Benefits**:
- **Code Reuse**: Common query patterns used across functions
- **Consistency**: Same logic used everywhere reduces bugs
- **Maintainability**: Changes to query logic only need to happen in one place

### 2. **Error Handling Patterns**

```typescript
export async function getCurrentUserOrThrow(ctx: QueryCtx) {
  const userRecord = await getCurrentUser(ctx);
  if (!userRecord) throw new Error("Can't get current user");
  return userRecord;
}
```

**Usage**: Functions that require authenticated users can use this helper instead of handling null checks everywhere.

## Performance Optimisation

### 1. **Query Efficiency**
```typescript
// Efficient: Uses index
.withIndex("byExternalId", (q) => q.eq("externalId", externalId))

// Less efficient: Full table scan
.filter((q) => q.eq(q.field("externalId"), externalId))
```

### 2. **Pagination Patterns**
```typescript
// For large datasets
export const paginatedPayments = query({
  args: { 
    cursor: v.optional(v.string()),
    limit: v.number()
  },
  handler: async (ctx, { cursor, limit }) => {
    return await ctx.db
      .query("paymentAttempts")
      .withIndex("byUserId")
      .paginate({ cursor, numItems: limit });
  },
});
```

## Development Best Practices

### 1. **Function Naming Conventions**
- **Queries**: `get*`, `list*`, `current`
- **Mutations**: `create*`, `update*`, `delete*`, `upsert*`  
- **Internal**: Prefix with purpose (`upsertFromClerk`, `savePaymentAttempt`)

### 2. **Error Handling**
```typescript
export const risky = mutation({
  handler: async (ctx, args) => {
    try {
      // Risky operation
      return result;
    } catch (error) {
      console.error("Operation failed:", error);
      throw new ConvexError("User-friendly error message");
    }
  },
});
```

### 3. **Testing Patterns**
- Use `npx convex dev` for local testing
- Test webhook endpoints with tools like ngrok
- Validate schema changes don't break existing queries

## Integration with Other Providers

### **Clerk Integration**
- JWT validation via `ctx.auth.getUserIdentity()`
- User sync via webhook handlers
- Payment processing via webhook transformations

### **Vercel Integration**  
- Deploy Convex functions independently of frontend
- Environment variables managed separately
- Functions accessible via generated URLs

## Next Steps

- [CLERK_INTEGRATION.md](./CLERK_INTEGRATION.md) - Authentication patterns
- [VERCEL_DEPLOYMENT.md](./VERCEL_DEPLOYMENT.md) - Deployment architecture  
- [BEST_PRACTICES.md](./BEST_PRACTICES.md) - Security and performance
- [INTEGRATION_SETUP.md](./INTEGRATION_SETUP.md) - Setup procedures