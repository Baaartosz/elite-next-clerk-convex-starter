# System Architecture Overview

## Introduction

This SaaS starter template implements a modern, serverless architecture that separates concerns across three primary external providers: **Clerk** (authentication & billing), **Convex** (real-time backend), and **Vercel** (hosting & edge functions). This document explains how these providers work together and why this architecture is effective for modern SaaS applications.

## High-Level Architecture

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Frontend      │    │   Authentication│    │    Backend      │
│   (Next.js 15)  │    │   & Billing     │    │   (Convex)      │
│                 │    │   (Clerk)       │    │                 │
│  • App Router   │◄──►│  • JWT Tokens   │◄──►│  • Real-time DB │
│  • Server Comp  │    │  • User Mgmt    │    │  • Functions    │
│  • Middleware   │    │  • Payments     │    │  • Webhooks     │
│  • Static Gen   │    │  • Sessions     │    │  • Validators   │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                       │                       │
         └───────────────────────┼───────────────────────┘
                                 ▼
                    ┌─────────────────────────┐
                    │     Deployment          │
                    │     (Vercel)            │
                    │                         │
                    │  • Edge Functions       │
                    │  • Static Hosting       │
                    │  • Environment Vars     │
                    │  • Domain Management    │
                    └─────────────────────────┘
```

## Provider Responsibilities

### 🎭 **Clerk** - Authentication & Billing Provider
**Role**: User identity, session management, and payment processing
**Responsibilities**:
- User authentication (social logins, email/password)
- Session management and JWT token generation
- Subscription billing and payment processing
- User profile management
- Organisational features
- Webhook events for user lifecycle

**Why Clerk**: Eliminates the complexity of building authentication and billing systems from scratch. Provides enterprise-grade security and compliance out of the box.

### 🗄️ **Convex** - Real-time Backend Provider  
**Role**: Database, serverless functions, and real-time synchronisation
**Responsibilities**:
- Real-time database with automatic sync
- Serverless function execution
- Schema validation and type safety
- Webhook handling
- Internal business logic
- Query optimisation and indexing

**Why Convex**: Provides real-time capabilities without websocket complexity. Type-safe throughout the stack with automatic client generation.

### 🚀 **Vercel** - Hosting & Edge Provider
**Role**: Frontend hosting, edge functions, and deployment
**Responsibilities**:
- Static site hosting with CDN
- Edge function execution
- Environment variable management  
- Domain management and SSL
- Build optimisation
- Analytics and monitoring

**Why Vercel**: Optimised for Next.js applications with zero-config deployments and global edge distribution.

## Data Flow Patterns

### 1. **User Authentication Flow**
```
User → Next.js Frontend → Clerk Auth → JWT Token → Convex Functions
  ↓                                                         ↓
Redirect to Dashboard ← Clerk Session ← User Data ← Database Query
```

### 2. **Real-time Data Synchronisation**
```
Frontend Component → useQuery(api.users.current)
       ↓
Convex Function → Database Query → Real-time Subscription
       ↓
Auto-update Frontend (no manual refresh needed)
```

### 3. **Payment Processing**
```
User → Clerk PricingTable → Clerk Billing → Payment Success
                                   ↓
                            Webhook to Convex → Database Update
                                   ↓  
                            Real-time UI Update (payment status)
```

### 4. **Webhook Integration**
```
Clerk Event → Webhook → Convex HTTP Handler → Internal Function → Database
    ↓                                                    ↓
User Change                                    Real-time Frontend Update
```

## Integration Boundaries

### **Frontend ↔ Clerk**
- **Connection**: `@clerk/nextjs` SDK
- **Data Exchange**: JWT tokens, user sessions, billing events
- **Communication**: HTTP requests, redirects, embedded components

### **Frontend ↔ Convex**  
- **Connection**: `convex/react` client with Clerk authentication
- **Data Exchange**: Queries, mutations, real-time subscriptions
- **Communication**: WebSocket for real-time, HTTP for functions

### **Clerk ↔ Convex**
- **Connection**: Webhooks and JWT validation
- **Data Exchange**: User events, payment events, authentication tokens
- **Communication**: HTTP webhooks, JWT token verification

### **Vercel ↔ All**
- **Connection**: Deployment platform and edge runtime
- **Data Exchange**: Environment variables, build artifacts
- **Communication**: Build processes, edge function execution

## Architecture Benefits

### **🔄 Real-time Synchronisation**
Changes propagate instantly across all connected clients without manual polling or complex WebSocket management.

### **🛡️ Security & Compliance**  
Clerk handles authentication security, JWT tokens, and PCI compliance for payments. Convex provides secure function execution with built-in auth validation.

### **📈 Scalability**
All providers scale automatically:
- Vercel handles traffic spikes with edge distribution
- Clerk scales authentication and billing
- Convex scales database and function execution

### **⚡ Developer Experience**
- Type safety across the entire stack
- Automatic client generation
- Real-time updates without complexity
- Zero-config deployments

### **🏗️ Separation of Concerns**
Each provider handles its specific domain expertise, reducing the complexity of any single system.

## System-Level Integration Points

### **JWT Token Flow**
1. Clerk generates JWT tokens with user identity
2. Next.js middleware validates tokens for protected routes
3. Convex validates JWT tokens for function access
4. Token contains user ID for database queries

### **Environment Configuration**  
- Shared JWT configuration between Clerk and Convex
- Webhook secrets for secure communication
- Environment-specific deployment URLs

### **Database Schema Synchronisation**
- Convex schema matches Clerk user structure
- Payment data structure matches Clerk billing webhooks
- Foreign key relationships via `externalId` fields

## Next Steps

This architecture overview provides the foundation. For detailed implementation guides, see:

- [CLERK_INTEGRATION.md](./CLERK_INTEGRATION.md) - Authentication and billing setup
- [CONVEX_BACKEND.md](./CONVEX_BACKEND.md) - Backend functions and database
- [VERCEL_DEPLOYMENT.md](./VERCEL_DEPLOYMENT.md) - Deployment and server-side options
- [BEST_PRACTICES.md](./BEST_PRACTICES.md) - Security and performance recommendations
- [INTEGRATION_SETUP.md](./INTEGRATION_SETUP.md) - Step-by-step setup guide