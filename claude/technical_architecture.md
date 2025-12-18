---
render_with_liquid: false
---

# Complete Technical Architecture & Implementation Guide

## Technology Stack Overview

### Backend

- **Runtime**: Bun.js (latest) - Fast, modern JavaScript runtime
- **Framework**: Elysia - High-performance web framework for Bun
- **Validation**: TypeBox - JSON Schema Type Builder with full type safety
- **Database**: PostgreSQL 16+ with connection pooling
- **ORM**: Drizzle ORM (latest) - Type-safe SQL ORM with Elysia integration
- **API Style**: RESTful + WebSocket for real-time updates

### Frontend - Web (Option 1)

- **Framework**: TanStack Start - SSR React framework
- **Form Management**: TanStack Form
- **Data Fetching**: TanStack Query (React Query)
- **Table/Grid**: TanStack Table
- **Routing**: File-based routing (built-in)
- **State Management**: Zustand / Jotai

### Frontend - Web/Mobile (Option 2)

- **Framework**: Expo (latest) - Universal app with file routing
- **Platform**: iOS, Android, Web
- **Navigation**: Expo Router (file-based)
- **Deployment**: EAS Build & EAS Update

### UI Components

- **DiceUI**: Modern, accessible components
- **reUI**: Beautiful, customizable components
- **shadcn/ui**: Radix-based, copy-paste components
- **Icons**: Lucide React
- **Styling**: Tailwind CSS

### Infrastructure

- **Database**: PostgreSQL (Supabase / Railway / Neon)
- **File Storage**: S3-compatible (Cloudflare R2 / MinIO)
- **Caching**: Redis (Upstash / Railway)
- **Monitoring**: Sentry, Axiom
- **Analytics**: PostHog / Plausible
- **Deployment**: Docker containers / Fly.io / Railway
- **CDN**: Cloudflare

## System Architecture

### High-Level Architecture

```
┌─────────────────── Client Layer ────────────────────┐
│                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────┐ │
│  │  Web App     │  │  Mobile App  │  │  Dashboard│ │
│  │  (TanStack   │  │  (Expo)      │  │  (Admin)  │ │
│  │   or Expo)   │  │              │  │           │ │
│  └──────────────┘  └──────────────┘  └───────────┘ │
│           │               │                 │       │
└───────────┼───────────────┼─────────────────┼───────┘
            │               │                 │
            └───────────────┴─────────────────┘
                            │
                  ┌─────────▼──────────┐
                  │   API Gateway      │
                  │   (Cloudflare)     │
                  └─────────┬──────────┘
                            │
            ┌───────────────┴────────────────┐
            │                                │
    ┌───────▼────────┐             ┌────────▼────────┐
    │  Application   │             │  WebSocket      │
    │  Server        │             │  Server         │
    │  (Bun/Elysia)  │◄───────────►│  (Bun/Elysia)   │
    └───────┬────────┘             └────────┬────────┘
            │                               │
    ┌───────┴────────┬──────────────────────┴───────┐
    │                │                              │
┌───▼────┐    ┌──────▼──────┐          ┌───────────▼────┐
│        │    │             │          │                │
│ Redis  │    │ PostgreSQL  │          │  File Storage  │
│ Cache  │    │ Database    │          │  (S3)          │
│        │    │ (Drizzle)   │          │                │
└────────┘    └──────┬──────┘          └────────────────┘
                     │
            ┌────────┴────────┐
            │                 │
    ┌───────▼──────┐  ┌──────▼────────┐
    │  Background  │  │  Message      │
    │  Jobs        │  │  Queue        │
    │  (BullMQ)    │  │  (Redis)      │
    └──────────────┘  └───────────────┘
```

### Microservices Architecture

```
┌─────────────────── Services Layer ──────────────────┐
│                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────┐ │
│  │   Booking    │  │   Payment    │  │   User    │ │
│  │   Service    │  │   Service    │  │  Service  │ │
│  └──────────────┘  └──────────────┘  └───────────┘ │
│                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────┐ │
│  │   Route      │  │  Notification│  │  Analytics│ │
│  │   Service    │  │   Service    │  │  Service  │ │
│  └──────────────┘  └──────────────┘  └───────────┘ │
│                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────┐ │
│  │   Partner    │  │   Pricing    │  │  Reviews  │ │
│  │   Service    │  │   Service    │  │  Service  │ │
│  └──────────────┘  └──────────────┘  └───────────┘ │
└──────────────────────────────────────────────────────┘
```

## Backend Architecture (Bun + Elysia + TypeBox + Drizzle)

### Project Structure

```
backend/
├── src/
│   ├── index.ts                 # Entry point
│   ├── config/
│   │   ├── env.ts               # Environment config
│   │   ├── database.ts          # DB config
│   │   └── redis.ts             # Cache config
│   ├── db/
│   │   ├── schema/              # Drizzle schemas
│   │   │   ├── users.ts
│   │   │   ├── bookings.ts
│   │   │   ├── routes.ts
│   │   │   ├── buses.ts
│   │   │   └── partners.ts
│   │   ├── migrations/          # DB migrations
│   │   └── index.ts             # DB instance
│   ├── modules/
│   │   ├── auth/
│   │   │   ├── auth.controller.ts
│   │   │   ├── auth.service.ts
│   │   │   ├── auth.schema.ts   # TypeBox schemas
│   │   │   └── auth.routes.ts
│   │   ├── bookings/
│   │   │   ├── booking.controller.ts
│   │   │   ├── booking.service.ts
│   │   │   ├── booking.schema.ts
│   │   │   └── booking.routes.ts
│   │   ├── routes/
│   │   ├── payments/
│   │   ├── users/
│   │   ├── partners/
│   │   └── analytics/
│   ├── common/
│   │   ├── middleware/
│   │   │   ├── auth.middleware.ts
│   │   │   ├── validation.middleware.ts
│   │   │   ├── error.middleware.ts
│   │   │   └── rate-limit.middleware.ts
│   │   ├── utils/
│   │   │   ├── password.ts
│   │   │   ├── jwt.ts
│   │   │   └── validators.ts
│   │   └── types/
│   │       └── index.ts
│   ├── services/
│   │   ├── email.service.ts
│   │   ├── sms.service.ts
│   │   ├── payment.service.ts
│   │   └── cache.service.ts
│   └── jobs/
│       ├── booking-confirmation.ts
│       ├── payment-processing.ts
│       └── analytics.ts
├── drizzle.config.ts
├── package.json
└── tsconfig.json
```

### Core Backend Implementation

#### 1. Database Schema (Drizzle + TypeBox)

```typescript
// src/db/schema/users.ts
import {
  pgTable,
  serial,
  varchar,
  timestamp,
  boolean,
  jsonb,
} from "drizzle-orm/pg-core";
import { Type, Static } from "@sinclair/typebox";

export const users = pgTable("users", {
  id: serial("id").primaryKey(),
  email: varchar("email", { length: 255 }).notNull().unique(),
  phone: varchar("phone", { length: 20 }).unique(),
  passwordHash: varchar("password_hash", { length: 255 }),
  firstName: varchar("first_name", { length: 100 }).notNull(),
  lastName: varchar("last_name", { length: 100 }).notNull(),
  role: varchar("role", { length: 20 }).notNull().default("customer"),
  isVerified: boolean("is_verified").notNull().default(false),
  preferences: jsonb("preferences"),
  createdAt: timestamp("created_at").notNull().defaultNow(),
  updatedAt: timestamp("updated_at").notNull().defaultNow(),
});

// TypeBox schema for validation
export const UserSchema = Type.Object({
  email: Type.String({ format: "email" }),
  phone: Type.Optional(Type.String({ pattern: "^\\+213[0-9]{9}$" })),
  firstName: Type.String({ minLength: 2, maxLength: 100 }),
  lastName: Type.String({ minLength: 2, maxLength: 100 }),
  password: Type.String({ minLength: 8 }),
});

export const UserLoginSchema = Type.Object({
  email: Type.String({ format: "email" }),
  password: Type.String(),
});

export type User = typeof users.$inferSelect;
export type NewUser = typeof users.$inferInsert;
export type UserDTO = Static<typeof UserSchema>;
```

```typescript
// src/db/schema/bookings.ts
import {
  pgTable,
  serial,
  integer,
  varchar,
  timestamp,
  decimal,
  jsonb,
  index,
} from "drizzle-orm/pg-core";
import { Type } from "@sinclair/typebox";

export const bookings = pgTable(
  "bookings",
  {
    id: serial("id").primaryKey(),
    bookingRef: varchar("booking_ref", { length: 20 }).notNull().unique(),
    userId: integer("user_id")
      .notNull()
      .references(() => users.id),
    routeId: integer("route_id")
      .notNull()
      .references(() => routes.id),
    departureTime: timestamp("departure_time").notNull(),
    arrivalTime: timestamp("arrival_time").notNull(),
    seatNumbers: jsonb("seat_numbers").notNull(),
    passengers: jsonb("passengers").notNull(),
    price: decimal("price", { precision: 10, scale: 2 }).notNull(),
    commission: decimal("commission", { precision: 10, scale: 2 }).notNull(),
    status: varchar("status", { length: 20 }).notNull().default("pending"),
    paymentStatus: varchar("payment_status", { length: 20 })
      .notNull()
      .default("pending"),
    paymentMethod: varchar("payment_method", { length: 50 }),
    ancillaries: jsonb("ancillaries"),
    createdAt: timestamp("created_at").notNull().defaultNow(),
    updatedAt: timestamp("updated_at").notNull().defaultNow(),
  },
  (table) => ({
    userIdx: index("user_idx").on(table.userId),
    routeIdx: index("route_idx").on(table.routeId),
    statusIdx: index("status_idx").on(table.status),
    bookingRefIdx: index("booking_ref_idx").on(table.bookingRef),
  })
);

export const BookingSchema = Type.Object({
  routeId: Type.Integer(),
  departureTime: Type.String({ format: "date-time" }),
  seatNumbers: Type.Array(Type.String()),
  passengers: Type.Array(
    Type.Object({
      firstName: Type.String(),
      lastName: Type.String(),
      age: Type.Optional(Type.Integer()),
      idNumber: Type.Optional(Type.String()),
    })
  ),
  paymentMethod: Type.Union([
    Type.Literal("cash"),
    Type.Literal("card"),
    Type.Literal("mobile_money"),
  ]),
  ancillaries: Type.Optional(
    Type.Object({
      extraLuggage: Type.Optional(Type.Boolean()),
      seatSelection: Type.Optional(Type.Boolean()),
      insurance: Type.Optional(Type.Boolean()),
    })
  ),
});

export type Booking = typeof bookings.$inferSelect;
export type NewBooking = typeof bookings.$inferInsert;
```

```typescript
// src/db/schema/routes.ts
import {
  pgTable,
  serial,
  integer,
  varchar,
  decimal,
  timestamp,
  time,
  boolean,
  jsonb,
} from "drizzle-orm/pg-core";

export const routes = pgTable("routes", {
  id: serial("id").primaryKey(),
  partnerId: integer("partner_id")
    .notNull()
    .references(() => partners.id),
  origin: varchar("origin", { length: 100 }).notNull(),
  destination: varchar("destination", { length: 100 }).notNull(),
  distance: decimal("distance", { precision: 6, scale: 2 }).notNull(),
  duration: integer("duration").notNull(), // minutes
  departureTime: time("departure_time").notNull(),
  arrivalTime: time("arrival_time").notNull(),
  frequency: varchar("frequency", { length: 20 }).notNull(), // daily, weekdays, etc.
  basePrice: decimal("base_price", { precision: 10, scale: 2 }).notNull(),
  totalSeats: integer("total_seats").notNull(),
  busType: varchar("bus_type", { length: 50 }).notNull(),
  amenities: jsonb("amenities"),
  isActive: boolean("is_active").notNull().default(true),
  createdAt: timestamp("created_at").notNull().defaultNow(),
  updatedAt: timestamp("updated_at").notNull().defaultNow(),
});

export const RouteSchema = Type.Object({
  origin: Type.String(),
  destination: Type.String(),
  distance: Type.Number(),
  duration: Type.Integer(),
  departureTime: Type.String(),
  basePrice: Type.Number(),
  totalSeats: Type.Integer({ minimum: 20, maximum: 60 }),
  busType: Type.Union([
    Type.Literal("standard"),
    Type.Literal("premium"),
    Type.Literal("luxury"),
  ]),
});
```

#### 2. Elysia API Implementation

```typescript
// src/index.ts
import { Elysia } from "elysia";
import { cors } from "@elysiajs/cors";
import { swagger } from "@elysiajs/swagger";
import { jwt } from "@elysiajs/jwt";
import { rateLimit } from "elysia-rate-limit";

import { authRoutes } from "./modules/auth/auth.routes";
import { bookingRoutes } from "./modules/bookings/booking.routes";
import { routeRoutes } from "./modules/routes/route.routes";
import { userRoutes } from "./modules/users/user.routes";
import { paymentRoutes } from "./modules/payments/payment.routes";
import { errorMiddleware } from "./common/middleware/error.middleware";

const app = new Elysia()
  .use(cors())
  .use(
    swagger({
      documentation: {
        info: {
          title: "Algeria Transportation Platform API",
          version: "1.0.0",
        },
        tags: [
          { name: "Auth", description: "Authentication endpoints" },
          { name: "Bookings", description: "Booking management" },
          { name: "Routes", description: "Route information" },
          { name: "Users", description: "User management" },
          { name: "Payments", description: "Payment processing" },
        ],
      },
    })
  )
  .use(
    jwt({
      name: "jwt",
      secret: process.env.JWT_SECRET!,
    })
  )
  .use(
    rateLimit({
      max: 100,
      duration: 60000, // 1 minute
    })
  )
  .onError(errorMiddleware)
  .group("/api/v1", (app) =>
    app
      .use(authRoutes)
      .use(bookingRoutes)
      .use(routeRoutes)
      .use(userRoutes)
      .use(paymentRoutes)
  )
  .listen(3000);

console.log(`🚀 Server running at ${app.server?.hostname}:${app.server?.port}`);
```

```typescript
// src/modules/auth/auth.routes.ts
import { Elysia, t } from "elysia";
import { UserSchema, UserLoginSchema } from "../../db/schema/users";
import { AuthService } from "./auth.service";

export const authRoutes = new Elysia({ prefix: "/auth" })
  .decorate("authService", new AuthService())
  .post(
    "/register",
    async ({ body, authService }) => {
      const user = await authService.register(body);
      return { success: true, data: user };
    },
    {
      body: UserSchema,
      detail: {
        tags: ["Auth"],
        summary: "Register new user",
      },
    }
  )
  .post(
    "/login",
    async ({ body, authService, jwt }) => {
      const user = await authService.login(body);
      const token = await jwt.sign({ userId: user.id, role: user.role });
      return {
        success: true,
        data: { user, token },
      };
    },
    {
      body: UserLoginSchema,
      detail: {
        tags: ["Auth"],
        summary: "User login",
      },
    }
  )
  .post(
    "/verify-otp",
    async ({ body, authService }) => {
      const result = await authService.verifyOTP(body.phone, body.otp);
      return { success: true, data: result };
    },
    {
      body: t.Object({
        phone: t.String(),
        otp: t.String({ minLength: 6, maxLength: 6 }),
      }),
      detail: {
        tags: ["Auth"],
        summary: "Verify OTP for phone verification",
      },
    }
  );
```

```typescript
// src/modules/bookings/booking.routes.ts
import { Elysia, t } from "elysia";
import { BookingSchema } from "../../db/schema/bookings";
import { BookingService } from "./booking.service";
import { authMiddleware } from "../../common/middleware/auth.middleware";

export const bookingRoutes = new Elysia({ prefix: "/bookings" })
  .decorate("bookingService", new BookingService())
  .use(authMiddleware)
  .get(
    "/",
    async ({ query, bookingService, user }) => {
      const bookings = await bookingService.getUserBookings(user.id, query);
      return { success: true, data: bookings };
    },
    {
      query: t.Object({
        page: t.Optional(t.Numeric()),
        limit: t.Optional(t.Numeric()),
        status: t.Optional(t.String()),
      }),
      detail: {
        tags: ["Bookings"],
        summary: "Get user bookings",
      },
    }
  )
  .get(
    "/:id",
    async ({ params, bookingService, user }) => {
      const booking = await bookingService.getBookingById(params.id, user.id);
      return { success: true, data: booking };
    },
    {
      params: t.Object({
        id: t.Numeric(),
      }),
      detail: {
        tags: ["Bookings"],
        summary: "Get booking by ID",
      },
    }
  )
  .post(
    "/",
    async ({ body, bookingService, user }) => {
      const booking = await bookingService.createBooking(user.id, body);
      return { success: true, data: booking };
    },
    {
      body: BookingSchema,
      detail: {
        tags: ["Bookings"],
        summary: "Create new booking",
      },
    }
  )
  .patch(
    "/:id/cancel",
    async ({ params, bookingService, user }) => {
      const result = await bookingService.cancelBooking(params.id, user.id);
      return { success: true, data: result };
    },
    {
      params: t.Object({
        id: t.Numeric(),
      }),
      detail: {
        tags: ["Bookings"],
        summary: "Cancel booking",
      },
    }
  );
```

#### 3. Service Layer with Drizzle ORM

```typescript
// src/modules/bookings/booking.service.ts
import { eq, and, desc, sql } from "drizzle-orm";
import { db } from "../../db";
import { bookings, routes, users } from "../../db/schema";
import { PricingService } from "../pricing/pricing.service";
import { NotificationService } from "../../services/notification.service";
import { CacheService } from "../../services/cache.service";

export class BookingService {
  private pricingService = new PricingService();
  private notificationService = new NotificationService();
  private cacheService = new CacheService();

  async createBooking(userId: number, data: BookingDTO) {
    // 1. Get route details
    const route = await db.query.routes.findFirst({
      where: eq(routes.id, data.routeId),
    });

    if (!route) {
      throw new Error("Route not found");
    }

    // 2. Check seat availability
    const availableSeats = await this.checkSeatAvailability(
      data.routeId,
      data.departureTime
    );

    if (availableSeats < data.seatNumbers.length) {
      throw new Error("Not enough seats available");
    }

    // 3. Calculate dynamic price
    const price = await this.pricingService.calculatePrice({
      routeId: data.routeId,
      basePrice: route.basePrice,
      seats: data.seatNumbers.length,
      departureTime: data.departureTime,
      ancillaries: data.ancillaries,
    });

    // 4. Calculate commission (27%)
    const commission = price * 0.27;

    // 5. Generate booking reference
    const bookingRef = this.generateBookingRef();

    // 6. Create booking
    const [booking] = await db
      .insert(bookings)
      .values({
        bookingRef,
        userId,
        routeId: data.routeId,
        departureTime: new Date(data.departureTime),
        arrivalTime: new Date(data.departureTime).setMinutes(
          new Date(data.departureTime).getMinutes() + route.duration
        ),
        seatNumbers: data.seatNumbers,
        passengers: data.passengers,
        price: price.toString(),
        commission: commission.toString(),
        status: "confirmed",
        paymentStatus: data.paymentMethod === "cash" ? "pending" : "processing",
        paymentMethod: data.paymentMethod,
        ancillaries: data.ancillaries || null,
      })
      .returning();

    // 7. Clear cache
    await this.cacheService.del(`route:${data.routeId}:availability`);

    // 8. Send confirmation
    await this.notificationService.sendBookingConfirmation(booking);

    return booking;
  }

  async getUserBookings(userId: number, options: QueryOptions) {
    const { page = 1, limit = 10, status } = options;
    const offset = (page - 1) * limit;

    const conditions = [eq(bookings.userId, userId)];
    if (status) {
      conditions.push(eq(bookings.status, status));
    }

    const userBookings = await db.query.bookings.findMany({
      where: and(...conditions),
      with: {
        route: true,
      },
      limit,
      offset,
      orderBy: [desc(bookings.createdAt)],
    });

    const [{ count }] = await db
      .select({ count: sql<number>`count(*)` })
      .from(bookings)
      .where(and(...conditions));

    return {
      data: userBookings,
      pagination: {
        page,
        limit,
        total: count,
        totalPages: Math.ceil(count / limit),
      },
    };
  }

  private async checkSeatAvailability(routeId: number, departureTime: string) {
    // Check cache first
    const cacheKey = `route:${routeId}:${departureTime}:availability`;
    const cached = await this.cacheService.get(cacheKey);

    if (cached) {
      return Number(cached);
    }

    // Query database
    const route = await db.query.routes.findFirst({
      where: eq(routes.id, routeId),
    });

    if (!route) return 0;

    const [{ bookedSeats }] = await db
      .select({ bookedSeats: sql<number>`count(*)` })
      .from(bookings)
      .where(
        and(
          eq(bookings.routeId, routeId),
          eq(bookings.departureTime, new Date(departureTime)),
          eq(bookings.status, "confirmed")
        )
      );

    const available = route.totalSeats - bookedSeats;

    // Cache for 5 minutes
    await this.cacheService.set(cacheKey, available.toString(), 300);

    return available;
  }

  private generateBookingRef(): string {
    const timestamp = Date.now().toString(36).toUpperCase();
    const random = Math.random().toString(36).substring(2, 6).toUpperCase();
    return `BK${timestamp}${random}`;
  }
}
```

#### 4. Dynamic Pricing Service

```typescript
// src/modules/pricing/pricing.service.ts
import { db } from "../../db";
import { bookings, routes } from "../../db/schema";
import { eq, and, gte, lte } from "drizzle-orm";

export class PricingService {
  async calculatePrice(params: PriceParams): Promise<number> {
    const { routeId, basePrice, seats, departureTime, ancillaries } = params;

    let finalPrice = Number(basePrice) * seats;

    // 1. Time-based multiplier
    const daysUntilDeparture = this.getDaysUntilDeparture(departureTime);
    const timeMultiplier = this.getTimeMultiplier(daysUntilDeparture);
    finalPrice *= timeMultiplier;

    // 2. Demand-based multiplier
    const demandMultiplier = await this.getDemandMultiplier(
      routeId,
      departureTime
    );
    finalPrice *= demandMultiplier;

    // 3. Day of week multiplier
    const dayMultiplier = this.getDayOfWeekMultiplier(departureTime);
    finalPrice *= dayMultiplier;

    // 4. Add ancillaries
    if (ancillaries) {
      if (ancillaries.extraLuggage) finalPrice += 200 * seats;
      if (ancillaries.seatSelection) finalPrice += 50 * seats;
      if (ancillaries.insurance) finalPrice += 150 * seats;
    }

    return Math.round(finalPrice);
  }

  private getDaysUntilDeparture(departureTime: string): number {
    const departure = new Date(departureTime);
    const now = new Date();
    const diff = departure.getTime() - now.getTime();
    return Math.ceil(diff / (1000 * 60 * 60 * 24));
  }

  private getTimeMultiplier(days: number): number {
    if (days >= 30) return 0.8; // 20% discount
    if (days >= 15) return 0.9; // 10% discount
    if (days >= 7) return 1.0; // base price
    if (days >= 2) return 1.15; // 15% increase
    if (days >= 1) return 1.3; // 30% increase
    return 1.4; // 40% increase (same day)
  }

  private async getDemandMultiplier(
    routeId: number,
    departureTime: string
  ): Promise<number> {
    const route = await db.query.routes.findFirst({
      where: eq(routes.id, routeId),
    });

    if (!route) return 1.0;

    const departure = new Date(departureTime);
    const dayStart = new Date(departure);
    dayStart.setHours(0, 0, 0, 0);
    const dayEnd = new Date(departure);
    dayEnd.setHours(23, 59, 59, 999);

    const [{ count }] = await db
      .select({ count: sql<number>`count(*)` })
      .from(bookings)
      .where(
        and(
          eq(bookings.routeId, routeId),
          gte(bookings.departureTime, dayStart),
          lte(bookings.departureTime, dayEnd),
          eq(bookings.status, "confirmed")
        )
      );

    const occupancyRate = count / route.totalSeats;

    if (occupancyRate >= 0.8) return 1.3; // 30% increase (high demand)
    if (occupancyRate >= 0.6) return 1.15; // 15% increase
    if (occupancyRate >= 0.4) return 1.0; // base price
    return 0.95; // 5% discount (low demand)
  }

  private getDayOfWeekMultiplier(departureTime: string): number {
    const day = new Date(departureTime).getDay();

    // Thursday evening = weekend start (high demand)
    if (day === 4) return 1.15;

    // Friday = low demand (holy day)
    if (day === 5) return 0.9;

    // Sunday = return travel (high demand)
    if (day === 0) return 1.1;

    return 1.0;
  }
}
```

## Frontend Architecture

### Option 1: TanStack Start (Web-Only)

```
tanstack-web/
├── app/
│   ├── routes/
│   │   ├── index.tsx              # Home page
│   │   ├── search.tsx             # Route search
│   │   ├── booking/
│   │   │   ├── $id.tsx            # Booking details
│   │   │   └── new.tsx            # New booking
│   │   ├── auth/
│   │   │   ├── login.tsx
│   │   │   └── register.tsx
│   │   ├── profile/
│   │   │   └── index.tsx
│   │   └── __root.tsx             # Root layout
│   ├── components/
│   │   ├── ui/                    # shadcn/ui components
│   │   ├── layout/
│   │   │   ├── Header.tsx
│   │   │   ├── Footer.tsx
│   │   │   └── Sidebar.tsx
│   │   ├── search/
│   │   │   ├── RouteSearchForm.tsx
│   │   │   ├── RouteCard.tsx
│   │   │   └── FilterSidebar.tsx
│   │   └── booking/
│   │       ├── SeatSelector.tsx
│   │       ├── PassengerForm.tsx
│   │       └── PaymentForm.tsx
│   ├── lib/
│   │   ├── api/
│   │   │   ├── client.ts
│   │   │   ├── routes.ts
│   │   │   ├── bookings.ts
│   │   │   └── auth.ts
│   │   ├── hooks/
│   │   │   ├── useAuth.ts
│   │   │   ├── useBooking.ts
│   │   │   └── useRoutes.ts
│   │   ├── utils/
│   │   │   ├── format.ts
│   │   │   └── validation.ts
│   │   └── store/
│   │       ├── auth.store.ts
│   │       └── booking.store.ts
│   └── styles/
│       └── globals.css
├── public/
├── package.json
└── tsconfig.json
```

#### TanStack Start Implementation

```typescript
// app/routes/index.tsx
import { createFileRoute } from "@tanstack/react-router";
import { useQuery } from "@tanstack/react-query";
import { RouteSearchForm } from "../components/search/RouteSearchForm";
import { PopularRoutes } from "../components/home/PopularRoutes";
import { Features } from "../components/home/Features";

export const Route = createFileRoute("/")({
  component: HomePage,
});

function HomePage() {
  const { data: popularRoutes } = useQuery({
    queryKey: ["routes", "popular"],
    queryFn: () => fetch("/api/v1/routes/popular").then((r) => r.json()),
  });

  return (
    <div className="min-h-screen">
      <section className="hero bg-gradient-to-br from-emerald-600 to-teal-700 text-white py-20">
        <div className="container mx-auto px-4">
          <h1 className="text-5xl font-bold mb-6">
            Travel Algeria with Comfort
          </h1>
          <p className="text-xl mb-8">
            Book your intercity bus journey in seconds
          </p>
          <RouteSearchForm />
        </div>
      </section>

      <PopularRoutes routes={popularRoutes?.data} />
      <Features />
    </div>
  );
}
```

```typescript
// app/components/search/RouteSearchForm.tsx
import { useForm } from "@tanstack/react-form";
import { useMutation } from "@tanstack/react-query";
import { useNavigate } from "@tanstack/react-router";
import { zodValidator } from "@tanstack/zod-form-adapter";
import { z } from "zod";
import { Button } from "../ui/button";
import { Input } from "../ui/input";
import { Label } from "../ui/label";
import { Calendar } from "../ui/calendar";

const searchSchema = z.object({
  origin: z.string().min(2, "Origin required"),
  destination: z.string().min(2, "Destination required"),
  date: z.date(),
  passengers: z.number().min(1).max(9),
});

export function RouteSearchForm() {
  const navigate = useNavigate();

  const form = useForm({
    defaultValues: {
      origin: "",
      destination: "",
      date: new Date(),
      passengers: 1,
    },
    onSubmit: async ({ value }) => {
      navigate({
        to: "/search",
        search: {
          from: value.origin,
          to: value.destination,
          date: value.date.toISOString(),
          passengers: value.passengers,
        },
      });
    },
    validatorAdapter: zodValidator,
  });

  return (
    <form
      onSubmit={(e) => {
        e.preventDefault();
        form.handleSubmit();
      }}
      className="bg-white rounded-lg shadow-lg p-6"
    >
      <div className="grid grid-cols-1 md:grid-cols-4 gap-4">
        <form.Field
          name="origin"
          validators={{
            onChange: searchSchema.shape.origin,
          }}
        >
          {(field) => (
            <div>
              <Label htmlFor={field.name}>From</Label>
              <Input
                id={field.name}
                value={field.state.value}
                onChange={(e) => field.handleChange(e.target.value)}
                placeholder="Algiers"
              />
              {field.state.meta.errors && (
                <span className="text-red-500 text-sm">
                  {field.state.meta.errors[0]}
                </span>
              )}
            </div>
          )}
        </form.Field>

        <form.Field
          name="destination"
          validators={{
            onChange: searchSchema.shape.destination,
          }}
        >
          {(field) => (
            <div>
              <Label htmlFor={field.name}>To</Label>
              <Input
                id={field.name}
                value={field.state.value}
                onChange={(e) => field.handleChange(e.target.value)}
                placeholder="Oran"
              />
              {field.state.meta.errors && (
                <span className="text-red-500 text-sm">
                  {field.state.meta.errors[0]}
                </span>
              )}
            </div>
          )}
        </form.Field>

        <form.Field name="date">
          {(field) => (
            <div>
              <Label htmlFor={field.name}>Date</Label>
              <Calendar
                mode="single"
                selected={field.state.value}
                onSelect={(date) => field.handleChange(date)}
                disabled={(date) => date < new Date()}
              />
            </div>
          )}
        </form.Field>

        <form.Field name="passengers">
          {(field) => (
            <div>
              <Label htmlFor={field.name}>Passengers</Label>
              <Input
                id={field.name}
                type="number"
                min="1"
                max="9"
                value={field.state.value}
                onChange={(e) => field.handleChange(Number(e.target.value))}
              />
            </div>
          )}
        </form.Field>
      </div>

      <Button type="submit" className="w-full mt-4" size="lg">
        Search Routes
      </Button>
    </form>
  );
}
```

```typescript
// app/routes/search.tsx
import { createFileRoute } from "@tanstack/react-router";
import { useQuery } from "@tanstack/react-query";
import { RouteCard } from "../components/search/RouteCard";
import { FilterSidebar } from "../components/search/FilterSidebar";

export const Route = createFileRoute("/search")({
  component: SearchPage,
  validateSearch: (search) => ({
    from: (search.from as string) || "",
    to: (search.to as string) || "",
    date: (search.date as string) || new Date().toISOString(),
    passengers: Number(search.passengers) || 1,
  }),
});

function SearchPage() {
  const { from, to, date, passengers } = Route.useSearch();

  const { data, isLoading } = useQuery({
    queryKey: ["routes", "search", from, to, date, passengers],
    queryFn: () =>
      fetch(
        `/api/v1/routes/search?from=${from}&to=${to}&date=${date}&passengers=${passengers}`
      ).then((r) => r.json()),
  });

  if (isLoading) {
    return <div>Loading routes...</div>;
  }

  return (
    <div className="container mx-auto px-4 py-8">
      <h1 className="text-3xl font-bold mb-6">
        {from} → {to}
      </h1>

      <div className="grid grid-cols-1 lg:grid-cols-4 gap-6">
        <div className="lg:col-span-1">
          <FilterSidebar />
        </div>

        <div className="lg:col-span-3 space-y-4">
          {data?.data?.map((route) => (
            <RouteCard key={route.id} route={route} passengers={passengers} />
          ))}

          {data?.data?.length === 0 && (
            <div className="text-center py-12">
              <p className="text-gray-500 text-lg">
                No routes found for your search criteria
              </p>
            </div>
          )}
        </div>
      </div>
    </div>
  );
}
```

### Option 2: Expo Universal (Web + Mobile)

```
expo-app/
├── app/
│   ├── (tabs)/
│   │   ├── _layout.tsx
│   │   ├── index.tsx              # Home
│   │   ├── search.tsx             # Search
│   │   ├── bookings.tsx           # My Bookings
│   │   └── profile.tsx            # Profile
│   ├── (auth)/
│   │   ├── login.tsx
│   │   └── register.tsx
│   ├── booking/
│   │   ├── [id].tsx               # Booking details
│   │   └── new.tsx                # Create booking
│   ├── route/
│   │   └── [id].tsx               # Route details
│   └── _layout.tsx                # Root layout
├── components/
│   ├── ui/                        # reUI / DiceUI components
│   ├── search/
│   ├── booking/
│   └── common/
├── lib/
│   ├── api/
│   ├── hooks/
│   ├── store/
│   └── utils/
├── constants/
│   ├── Colors.ts
│   └── Layout.ts
├── assets/
├── app.json
├── package.json
└── tsconfig.json
```

#### Expo Implementation

```typescript
// app/(tabs)/_layout.tsx
import { Tabs } from "expo-router";
import { Home, Search, List, User } from "lucide-react-native";

export default function TabLayout() {
  return (
    <Tabs
      screenOptions={{
        tabBarActiveTintColor: "#10b981",
        headerShown: false,
      }}
    >
      <Tabs.Screen
        name="index"
        options={{
          title: "Home",
          tabBarIcon: ({ color }) => <Home size={24} color={color} />,
        }}
      />
      <Tabs.Screen
        name="search"
        options={{
          title: "Search",
          tabBarIcon: ({ color }) => <Search size={24} color={color} />,
        }}
      />
      <Tabs.Screen
        name="bookings"
        options={{
          title: "My Bookings",
          tabBarIcon: ({ color }) => <List size={24} color={color} />,
        }}
      />
      <Tabs.Screen
        name="profile"
        options={{
          title: "Profile",
          tabBarIcon: ({ color }) => <User size={24} color={color} />,
        }}
      />
    </Tabs>
  );
}
```

```typescript
// app/(tabs)/index.tsx
import { View, ScrollView, StyleSheet } from "react-native";
import { useQuery } from "@tanstack/react-query";
import { RouteSearchForm } from "../../components/search/RouteSearchForm";
import { PopularRoutes } from "../../components/home/PopularRoutes";
import { Text } from "../../components/ui/Text";
import { Card } from "../../components/ui/Card";

export default function HomeScreen() {
  const { data: popularRoutes } = useQuery({
    queryKey: ["routes", "popular"],
    queryFn: () => fetch("/api/v1/routes/popular").then((r) => r.json()),
  });

  return (
    <ScrollView style={styles.container}>
      <View style={styles.hero}>
        <Text style={styles.title}>Travel Algeria with Comfort</Text>
        <Text style={styles.subtitle}>
          Book your intercity bus journey in seconds
        </Text>
      </View>

      <RouteSearchForm />

      <View style={styles.section}>
        <Text style={styles.sectionTitle}>Popular Routes</Text>
        <PopularRoutes routes={popularRoutes?.data} />
      </View>
    </ScrollView>
  );
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
    backgroundColor: "#fff",
  },
  hero: {
    padding: 24,
    backgroundColor: "#10b981",
  },
  title: {
    fontSize: 32,
    fontWeight: "bold",
    color: "#fff",
    marginBottom: 8,
  },
  subtitle: {
    fontSize: 18,
    color: "#f3f4f6",
  },
  section: {
    padding: 16,
  },
  sectionTitle: {
    fontSize: 24,
    fontWeight: "bold",
    marginBottom: 16,
  },
});
```

## Dashboard Application (Admin Panel)

```
dashboard/
├── app/
│   ├── routes/
│   │   ├── dashboard/
│   │   │   ├── index.tsx          # Overview
│   │   │   ├── routes.tsx         # Route management
│   │   │   ├── bookings.tsx       # Booking management
│   │   │   ├── partners.tsx       # Partner management
│   │   │   ├── users.tsx          # User management
│   │   │   ├── analytics.tsx      # Analytics
│   │   │   └── settings.tsx       # Settings
│   │   └── login.tsx
│   └── components/
│       ├── dashboard/
│       │   ├── Sidebar.tsx
│       │   ├── Header.tsx
│       │   ├── StatsCard.tsx
│       │   └── Charts.tsx
│       └── tables/
│           ├── BookingsTable.tsx
│           ├── RoutesTable.tsx
│           └── PartnersTable.tsx
```

## Deployment Architecture

### Docker Compose (Development)

```yaml
# docker-compose.yml
version: "3.8"

services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: transport_db
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: password
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data

  backend:
    build: ./backend
    ports:
      - "3000:3000"
    environment:
      DATABASE_URL: postgresql://admin:password@postgres:5432/transport_db
      REDIS_URL: redis://redis:6379
      JWT_SECRET: your-secret-key
    depends_on:
      - postgres
      - redis

  web:
    build: ./tanstack-web
    ports:
      - "3001:3000"
    environment:
      VITE_API_URL: http://localhost:3000

volumes:
  postgres_data:
  redis_data:
```

### Production Deployment (Fly.io / Railway)

```toml
# fly.toml
app = "algeria-transport-api"

[build]
  builder = "paketobuildpacks/builder:base"

[env]
  PORT = "3000"

[services]
  [[services.ports]]
    port = 80
    handlers = ["http"]
    force_https = true

  [[services.ports]]
    port = 443
    handlers = ["tls", "http"]

[[services.tcp_checks]]
  interval = "15s"
  timeout = "2s"
  grace_period = "1s"
```

## Performance Optimization

### 1. Database Optimization

- Connection pooling (pg-pool)
- Query optimization with indexes
- Read replicas for analytics
- Materialized views for reports

### 2. Caching Strategy

- Redis for session data
- Route availability cache (5 min TTL)
- Popular routes cache (1 hour TTL)
- CDN for static assets

### 3. Frontend Optimization

- Code splitting
- Image optimization (next/image or expo-image)
- Lazy loading
- Service Worker (PWA)

### 4. API Optimization

- Response compression (gzip)
- Rate limiting
- Request batching
- GraphQL (optional for complex queries)

---

**Continue to Part 2 for AI Integration Strategy...**
