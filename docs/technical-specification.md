# tadarus.my Technical Specification

## Overview
This document outlines the technical architecture and implementation details for tadarus.my (tdrsmy), a web application for Malaysian Muslims to track group Quran reading (tadarus) during Ramadan.

## Architecture

### Tech Stack
- **Frontend**: Next.js 14 (App Router) with React 19
- **Server**: Next.js Server Actions and React Server Components
- **UI**: Shadcn/UI components with Tailwind CSS
- **State Management**: React Context API + React Hooks
- **Database**: Drizzle ORM with Vercel Postgres
- **Authentication**: Clerk
- **Real-time Updates**: Pusher
- **Deployment**: Vercel (live at tadarus.my)
- **Runtime**: Bun
- **Form Validation**: Zod

### System Architecture
```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│                 │     │                 │     │                 │
│  Next.js App    │────▶│  Server Actions │────▶│  Vercel Postgres│
│  (React)        │     │  (Server)       │     │  (Database)     │
│                 │     │                 │     │                 │
└─────────────────┘     └─────────────────┘     └─────────────────┘
         │                      │                        │
         │                      │                        │
         ▼                      ▼                        │
┌─────────────────┐     ┌─────────────────┐             │
│                 │     │                 │             │
│  Clerk          │     │  Pusher         │◀────────────┘
│  (Auth)         │     │  (Real-time)    │
│                 │     │                 │
└─────────────────┘     └─────────────────┘
```

## Database Schema

### Tables

#### Users
```typescript
export const users = pgTable("users", {
  id: text("id").primaryKey(), // Clerk user ID
  name: text("name").notNull(),
  email: text("email").notNull().unique(),
  imageUrl: text("image_url"),
  createdAt: timestamp("created_at").defaultNow().notNull(),
  updatedAt: timestamp("updated_at").defaultNow().notNull(),
});

export type User = InferModel<typeof users>;
```

#### Circles (Tadarus Groups)
```typescript
export const tadarusCircleStatus = {
  ACTIVE: "active",
  COMPLETED: "completed",
  ARCHIVED: "archived",
} as const;

export const tadarusCircles = pgTable("tadarus_circles", {
  id: uuid("id").defaultRandom().primaryKey(),
  name: text("name").notNull(),
  description: text("description"),
  inviteCode: text("invite_code").notNull().unique(),
  targetPages: integer("target_pages").notNull().default(604), // Default: One Quran
  createdById: text("created_by_id").notNull().references(() => users.id, { onDelete: "cascade" }),
  status: text("status").notNull().default(tadarusCircleStatus.ACTIVE),
  createdAt: timestamp("created_at").defaultNow().notNull(),
  updatedAt: timestamp("updated_at").defaultNow().notNull(),
});

export type TadarusCircle = InferModel<typeof tadarusCircles>;
```

#### Circle Members
```typescript
export const circleMembers = pgTable("circle_members", {
  id: uuid("id").defaultRandom().primaryKey(),
  userId: text("user_id").notNull().references(() => users.id, { onDelete: "cascade" }),
  circleId: uuid("circle_id").notNull().references(() => tadarusCircles.id, { onDelete: "cascade" }),
  role: text("role").notNull().default("member"), // "admin" or "member"
  joinedAt: timestamp("joined_at").defaultNow().notNull(),
  updatedAt: timestamp("updated_at").defaultNow().notNull(),
});

export type CircleMember = InferModel<typeof circleMembers>;
```

#### Tadarus Logs
```typescript
export const tadarusLogs = pgTable("tadarus_logs", {
  id: uuid("id").defaultRandom().primaryKey(),
  userId: text("user_id").notNull().references(() => users.id, { onDelete: "cascade" }),
  circleId: uuid("circle_id").notNull().references(() => tadarusCircles.id, { onDelete: "cascade" }),
  pagesRead: integer("pages_read").notNull(),
  notes: text("notes"),
  createdAt: timestamp("created_at").defaultNow().notNull(),
  updatedAt: timestamp("updated_at").defaultNow().notNull(),
});

export type TadarusLog = InferModel<typeof tadarusLogs>;
```

#### Tadarus Wrapped
```typescript
export const tadarusWrapped = pgTable("tadarus_wrapped", {
  id: uuid("id").defaultRandom().primaryKey(),
  userId: text("user_id").notNull().references(() => users.id, { onDelete: "cascade" }),
  circleId: uuid("circle_id").notNull().references(() => tadarusCircles.id, { onDelete: "cascade" }),
  totalPagesRead: integer("total_pages_read").notNull(),
  userPagesRead: integer("user_pages_read").notNull(),
  percentageContribution: decimal("percentage_contribution", { precision: 5, scale: 2 }).notNull(),
  rank: integer("rank").notNull(),
  imageUrl: text("image_url"),
  createdAt: timestamp("created_at").defaultNow().notNull(),
});

export type TadarusWrapped = InferModel<typeof tadarusWrapped>;
```

## Server Actions

### Authentication
- Authentication handled by Clerk

### Tadarus Circles
- `getUserCircles` - Get all circles for the current user
- `createCircle` - Create a new circle
- `getCircleById` - Get circle details
- `updateCircle` - Update circle details
- `deleteCircle` - Delete a circle
- `joinCircle` - Join a circle with invite code

### Tadarus Logs
- `getUserLogs` - Get all logs for the current user
- `getCircleLogs` - Get all logs for a specific circle
- `createLog` - Create a new tadarus log
- `updateLog` - Update a log
- `deleteLog` - Delete a log

### Leaderboard
- `getLeaderboard` - Get top 10 circles by pages read

### Tadarus Wrapped
- `getWrappedData` - Get wrapped data for a circle
- `generateWrapped` - Generate wrapped data for a circle

## Server Actions Implementation

Server actions will be implemented using Next.js 14's built-in server actions feature. These actions will be defined in separate files within the app directory structure, following a domain-driven approach:

```
app/
├── actions/
│   ├── circles.ts
│   ├── logs.ts
│   ├── leaderboard.ts
│   └── wrapped.ts
```

Each action will be defined as an async function with the `"use server"` directive at the top of the file or function. For example:

```typescript
// app/actions/circles.ts
"use server";

import { auth } from "@clerk/nextjs";
import { db } from "@/lib/db";
import { tadarusCircles } from "@/lib/db/schema";
import { revalidatePath } from "next/cache";
import { z } from "zod";

const createCircleSchema = z.object({
  name: z.string().min(3).max(50),
  description: z.string().optional(),
  targetPages: z.number().int().positive().default(604),
});

export async function createCircle(formData: FormData) {
  const { userId } = auth();
  
  if (!userId) {
    throw new Error("Unauthorized");
  }
  
  const validatedFields = createCircleSchema.parse({
    name: formData.get("name"),
    description: formData.get("description"),
    targetPages: Number(formData.get("targetPages")),
  });
  
  // Generate a unique invite code
  const inviteCode = generateInviteCode();
  
  const circle = await db.insert(tadarusCircles).values({
    name: validatedFields.name,
    description: validatedFields.description,
    targetPages: validatedFields.targetPages,
    inviteCode,
    createdById: userId,
  }).returning();
  
  // Revalidate the dashboard page to show the new circle
  revalidatePath("/dashboard");
  
  return circle[0];
}
```

These server actions can be imported directly into client components and used with forms:

```tsx
// app/circles/new/page.tsx
"use client";

import { createCircle } from "@/app/actions/circles";

export default function NewCirclePage() {
  return (
    <form action={createCircle}>
      <input name="name" placeholder="Circle Name" />
      <textarea name="description" placeholder="Description" />
      <input name="targetPages" type="number" defaultValue={604} />
      <button type="submit">Create Circle</button>
    </form>
  );
}
```

Benefits of using server actions:
- Simplified data fetching and mutations
- Built-in CSRF protection
- Automatic form validation with Zod
- Seamless integration with React Server Components
- Reduced client-server code boundaries
- Improved performance with automatic cache invalidation

## Frontend Structure

### Pages
- `/` - Homepage with app introduction
- `/dashboard` - User dashboard showing their circles
- `/circles/new` - Create a new circle
- `/circles/join` - Join an existing circle
- `/circles/[id]` - Circle details page
- `/circles/[id]/logs` - Circle logs page
- `/leaderboard` - Global leaderboard
- `/wrapped/[circleId]` - Tadarus Wrapped page

### Components
- `CircleCard` - Display circle information
- `CreateCircleForm` - Form to create a new circle
- `JoinCircleForm` - Form to join a circle with invite code
- `LogEntryForm` - Form to log tadarus pages
- `ProgressBar` - Visual representation of circle progress
- `LeaderboardTable` - Display top circles
- `WrappedCard` - Shareable Tadarus Wrapped card

## Real-time Updates
Pusher will be used for real-time updates in the following scenarios:
- When a new log is added to a circle
- When the leaderboard changes
- When a circle reaches its target

## Authentication & Authorization
- Clerk will handle user authentication
- Circle creators are automatically assigned the "admin" role
- Only admins can modify circle details
- Any member can add logs to their circle

## Deployment Strategy
- CI/CD pipeline with GitHub Actions
- Automatic deployments to Vercel
- Database migrations handled by Drizzle ORM

## Performance Considerations
- Server-side rendering for initial page loads
- Server actions for data mutations with automatic cache invalidation
- React Server Components for reduced client-side JavaScript
- Optimistic UI updates for better user experience
- Streaming responses for improved time-to-first-byte
- Caching strategies for frequently accessed data
- Parallel data fetching with React Suspense

## Security Considerations
- All server actions protected with authentication
- Server actions validated with Zod schemas
- Built-in CSRF protection with Next.js server actions
- Rate limiting for sensitive operations
- Input validation using Zod

## Monitoring & Analytics
- Vercel Analytics for performance monitoring
- Custom event tracking for user actions
- Error logging with Sentry

## Future Enhancements
- Mobile app with React Native
- Push notifications
- Social sharing integrations
- Advanced analytics for tadarus patterns 