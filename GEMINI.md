# GEMINI.md — LLM Rules for E-course Platform
# =============================================================================
# This file is the single source of behavioral rules for any LLM assisting
# with development of this project inside Cursor or any AI-assisted IDE.
# Every code generation, refactor, or suggestion MUST comply with these rules.
# =============================================================================

## 1. Project Identity

- **Product:** E-course website with public marketing site, student learning portal, and admin dashboard.
- **Stack:** Next.js 16+ (App Router), TypeScript, Tailwind CSS, shadcn/ui, PostgreSQL, Prisma ORM.
- **Services:** Clerk (auth), Stripe (payments), Mux (video), Resend (email), Inngest (background jobs), PostHog (analytics).
- **Architecture:** Single codebase, single PostgreSQL database, server-side route/role protection.

---

## 2. Source of Truth — Required Reading

Before generating ANY code, you MUST reference and obey these project documents. If a document contradicts your training data, the document wins.

| Document                          | Purpose                                                  |
|-----------------------------------|----------------------------------------------------------|
| `Ecourse-platform-prd.md`        | Product requirements, scope, user stories, MVP definition |
| `ecourse_platform_db_schema.md`  | Database design, constraints, business rules, enums       |
| `schema.prisma`                  | Exact Prisma schema — the implementation truth            |
| `folder-architecture.md`         | File and folder structure for the entire codebase         |
| `site-map.md`                    | Routes, page purposes, sections, MVP page list            |
| `permissions-matrix.md`          | Role-based access control for routes and actions          |

**Rule:** If you are unsure about a table, field, enum, route, permission, or feature — look it up in the documents above. Do NOT guess.

---

## 3. Anti-Hallucination Rules

### 3.1 Do NOT invent
- Do NOT create tables, columns, or enums that do not exist in `schema.prisma` or `ecourse_platform_db_schema.md`.
- Do NOT add routes that are not in `site-map.md` or `folder-architecture.md`.
- Do NOT invent features, flows, or UI pages that are not in `Ecourse-platform-prd.md`.
- Do NOT add roles beyond: `SUPER_ADMIN`, `ADMIN`, `COURSE_MANAGER`, `SUPPORT`, `INSTRUCTOR`, `STUDENT`.
- Do NOT add order statuses, enrollment statuses, or any enum values beyond those defined in `schema.prisma`.

### 3.2 Do NOT assume
- Do NOT assume a field exists — verify it in `schema.prisma` first.
- Do NOT assume a relation exists — check the model's relations block.
- Do NOT assume a route exists — check `folder-architecture.md`.
- Do NOT assume a role can perform an action — check `permissions-matrix.md`.
- Do NOT assume subscriptions, quizzes, community features, or affiliate systems exist — they are post-MVP.

### 3.3 Do NOT over-engineer
- Do NOT build abstractions for features that are not in the MVP scope.
- Do NOT add caching layers, message queues, or microservice patterns unless explicitly asked.
- Do NOT create custom auth logic — Clerk handles authentication.
- Do NOT create custom payment processing — Stripe handles payments.
- Do NOT create custom video infrastructure — Mux handles video.

---

## 4. Database Rules

### 4.1 Schema compliance
- The Prisma schema in `schema.prisma` is the exact implementation. Use it as-is.
- All UUIDs use `String @id @default(uuid())`.
- All timestamps use `DateTime @default(now())` for createdAt and `@updatedAt` for updatedAt.
- Use the exact enum names and values defined in the schema. Do not rename or alias them.

### 4.2 Required constraints (enforce in application logic)
- `totalAmount = subtotalAmount - discountAmount + taxAmount` on Order.
- `totalAmount = quantity * unitAmount` on OrderItem.
- `progressPercent` must be 0–100 on LessonProgress and CourseProgress.
- `completedLessons <= totalLessons` on CourseProgress.
- PERCENTAGE coupon `amount` must be 1–100.
- `lesson.courseId` must match `section.courseId` — validate on every write.
- `lessonProgress.lessonId` must belong to `enrollment.courseId` — validate on every write.

### 4.3 Deletion policy
- **NEVER hard-delete:** Users with orders/enrollments, courses with orders/enrollments, orders, certificates, coupon redemptions.
- **Soft-delete allowed:** Use status enums (`ARCHIVED`, `DELETED`, `SUSPENDED`) or `isActive = false`.
- **Hard-delete allowed only for:** Draft-only content with zero historical references, accidental unlinked assets.

### 4.4 State transition rules
- If `Order.status = PAID` → `paidAt` must NOT be null.
- If `Course.status = PUBLISHED` → `publishedAt` must NOT be null.
- If `Course.status = ARCHIVED` → `archivedAt` must NOT be null.
- If `Lesson.isPublished = true` → `publishedAt` must NOT be null.
- If `Enrollment.status = COMPLETED` → `completedAt` must NOT be null.
- If `Enrollment.status = REVOKED` → `accessRevokedAt` must NOT be null.
- If `LessonProgress.status = COMPLETED` → `completedAt` must NOT be null.
- If `Refund.status = PROCESSED` → `processedAt` must NOT be null.

---

## 5. Folder Structure Rules

Follow `folder-architecture.md` exactly. Key principles:

```
src/
  app/
    (marketing)/    → Public pages: /, /courses, /courses/[slug], /checkout/[productId], /about, /faq, /contact
    (auth)/         → /sign-in, /sign-up (Clerk catch-all routes)
    (student)/      → /learn/* (student portal, requires auth + enrollment)
    (admin)/        → /admin/* (admin dashboard, requires auth + admin role)
    api/            → Webhook routes (stripe, clerk, mux) + REST endpoints
  components/
    ui/             → shadcn/ui base components
    marketing/      → Public website components
    student/        → Student portal components
    admin/          → Admin dashboard components
    shared/         → Cross-surface shared components
  features/         → Domain logic, hooks, helpers per feature
  lib/              → Third-party service clients and utilities
  server/
    actions/        → Next.js server actions
    queries/        → Database read queries
    services/       → Business logic services
    validators/     → Zod input validation schemas
  prisma/           → schema.prisma, migrations, seed.ts
  types/            → Shared TypeScript types
  constants/        → Role codes, payment config, enrollment sources, etc.
```

### Placement rules
- Server-only code goes in `src/server/`. Never import server code on the client.
- Prisma client instance lives in `src/lib/db/`.
- Permission checks live in `src/lib/permissions/`.
- Zod validators live in `src/server/validators/`.
- Each route group has its own `layout.tsx` for navigation and protection.

---

## 6. Route and Permission Rules

### 6.1 Route groups
- `(marketing)` — public, no auth required.
- `(auth)` — Clerk sign-in/sign-up, no auth required.
- `(student)` — requires authenticated user with STUDENT role and active enrollment for course-specific pages.
- `(admin)` — requires authenticated user with admin-level role (SUPER_ADMIN, ADMIN, COURSE_MANAGER, or SUPPORT depending on route).

### 6.2 Middleware rules
- Protect all `/learn/*` routes — redirect unauthenticated to `/sign-in`.
- Protect all `/admin/*` routes — redirect unauthenticated to `/sign-in`.
- Do NOT enforce role-level checks in middleware — handle in server actions and queries.

### 6.3 Permission enforcement
- Every server action that mutates data must call a permission check BEFORE executing.
- Use the permission helper pattern from `permissions-matrix.md`:
  ```ts
  requireRole(userId, [RoleCode.ADMIN, RoleCode.SUPER_ADMIN])
  ```
- Student enrollment access must query the DB for `Enrollment.status = ACTIVE`.
- REVOKED, REFUNDED, or EXPIRED enrollments block access at the server level — never rely on UI alone.

### 6.4 Exact permission mapping
Reference `permissions-matrix.md` for the full route and action matrices. Key rules:
- COURSE_MANAGER can manage courses/lessons/coupons but CANNOT view students, orders, analytics, or settings.
- SUPPORT can view students, orders, certificates but CANNOT manage courses, coupons, analytics, or settings.
- STUDENT can only access `/learn/*` routes and only their own enrolled content.
- INSTRUCTOR role is display-only in V1 — no login portal.

---

## 7. Authentication Rules (Clerk)

- Clerk handles all authentication. Do NOT build custom auth flows.
- Every user has a `clerkUserId` in the `User` table that maps to their Clerk identity.
- On Clerk webhook `user.created` → create a `User` record in the database with `STUDENT` role.
- On Clerk webhook `user.updated` → sync email and profile changes.
- Session validation uses Clerk's `auth()` helper in server components and `getAuth()` in API routes.
- Store app-level roles in the `UserRole` table, NOT in Clerk metadata.

---

## 8. Payment Rules (Stripe)

- Stripe handles all payment processing. Do NOT build custom payment logic.
- Checkout uses Stripe Checkout Sessions.
- Order creation flow:
  1. Create a PENDING order in the database.
  2. Create a Stripe Checkout Session.
  3. Store `stripeCheckoutSessionId` on the order.
  4. On `checkout.session.completed` webhook → update order to PAID + create enrollment.
- Webhook idempotency: Check `stripeCheckoutSessionId` before creating — skip if order already PAID.
- Wrap order status update + enrollment creation in a single Prisma transaction.
- Frontend success screens must NOT be trusted as the final source of payment truth.
- Never delete orders — use status transitions only.

---

## 9. Video Rules (Mux)

- Mux handles all video delivery.
- Videos are stored as `muxAssetId` and `muxPlaybackId` on the `Lesson` model.
- Use Mux webhooks to update asset processing status.
- For VIDEO-type lessons, `muxPlaybackId` is required for playback.
- Use Mux Player component for video rendering.

---

## 10. Business Logic Rules

### 10.1 Enrollment flow
- Successful PAID order → create enrollment with `enrollmentSource = PURCHASE`.
- Admin manual enrollment → `enrollmentSource = MANUAL`, MUST log to AuditLog with `action = "MANUAL_ENROLLMENT"`.
- One active enrollment per user per course (`@@unique([userId, courseId])`).

### 10.2 Progress tracking
- `LessonProgress` is the source of truth for lesson-level progress.
- `CourseProgress` is a cached aggregate — update it when `LessonProgress` changes.
- V1 completion rule: ALL published lessons must be completed for course completion.
- Completion trigger: `CourseProgress.completedLessons = CourseProgress.totalLessons`.

### 10.3 Certificate issuance
- Only issue when: `Enrollment.status = COMPLETED` AND `Course.hasCertificate = true`.
- One certificate per enrollment maximum (`@@unique([enrollmentId])`).
- Generate unique `certificateNumber` and `verificationCode`.

### 10.4 Coupon validation
- Coupon must have `isActive = true`.
- Current time must be within `startsAt` and `expiresAt` if set.
- `redeemedCount` must be less than `maxRedemptions` if set.
- PERCENTAGE type: amount must be 1–100.
- FIXED_AMOUNT type: amount is in smallest currency unit (cents).

---

## 11. Audit Log Rules

The following actions MUST always write to the `AuditLog` table:

| Action                  | entityType     | action constant           |
|-------------------------|----------------|---------------------------|
| Manual enrollment       | `"Enrollment"` | `"MANUAL_ENROLLMENT"`     |
| Revoke enrollment       | `"Enrollment"` | `"REVOKE_ACCESS"`         |
| Role assigned to user   | `"UserRole"`   | `"ROLE_ASSIGNED"`         |
| Role removed from user  | `"UserRole"`   | `"ROLE_REMOVED"`          |
| Course published        | `"Course"`     | `"COURSE_PUBLISHED"`      |
| Course archived         | `"Course"`     | `"COURSE_ARCHIVED"`       |
| Coupon deactivated      | `"Coupon"`     | `"COUPON_DEACTIVATED"`    |
| Manual order recovery   | `"Order"`      | `"MANUAL_ORDER_RECOVERY"` |

---

## 12. Coding Standards

### 12.1 TypeScript
- Strict mode enabled. No `any` types unless truly unavoidable.
- Use Zod for all input validation in `src/server/validators/`.
- Use explicit return types on server actions and queries.
- Prefer `interface` for object shapes, `type` for unions and intersections.

### 12.2 React / Next.js
- Use Server Components by default. Add `"use client"` only when client interactivity is required.
- Use Next.js server actions for mutations (forms, state changes).
- Use `src/server/queries/` for data fetching in server components.
- Use `src/server/actions/` for mutations triggered by user interaction.
- Wrap sensitive operations in `try/catch` with meaningful error handling.

### 12.3 Styling
- Use Tailwind CSS utility classes.
- Use shadcn/ui components from `src/components/ui/`.
- Do NOT install additional UI libraries unless explicitly requested.
- Keep component-specific styles inside the component file.

### 12.4 Naming conventions
- Files: `kebab-case.ts`, `kebab-case.tsx`.
- Components: `PascalCase`.
- Functions/variables: `camelCase`.
- Constants: `UPPER_SNAKE_CASE`.
- Database enums: `UPPER_SNAKE_CASE` (matching Prisma enum values).
- Routes: `kebab-case` (matching Next.js conventions).

### 12.5 Imports
- Use `@/` path alias for `src/` directory.
- Group imports: (1) external packages, (2) internal modules, (3) relative imports.
- Never import from `src/server/` in client components.

---

## 13. MVP Scope Guard

### MVP tables (build these first):
User, Role, UserRole, CourseCategory, Course, CourseSection, Lesson, LessonAsset, Product, Price, Order, OrderItem, Coupon, Enrollment, LessonProgress, CourseProgress, Certificate, AuditLog.

### Post-MVP tables (do NOT build unless explicitly asked):
CouponRedemption, Refund, Subscription, Notification, AdminNote.

### Post-MVP features (do NOT build unless explicitly asked):
- Subscriptions / recurring billing
- Community / discussions / Q&A
- Support ticketing system
- Instructor-specific dashboards
- Advanced analytics segmentation
- Affiliate program
- Quiz grading workflows
- Multi-language localization
- Native mobile apps
- Gamification systems
- AI personalization
- Blog CMS
- Drip content scheduling

If a feature belongs to post-MVP, say so and stop. Do not implement it.

---

## 14. MVP Page Build Order

Follow this order for the most stable incremental build:

### Phase 1 — Shell and routing
Layouts, route groups, navigation, auth protection, placeholder pages:
`/`, `/courses`, `/courses/[slug]`, `/sign-in`, `/sign-up`, `/learn`, `/admin`

### Phase 2 — Commerce and access
Checkout, order creation, enrollment, protected student access:
`/checkout/[productId]`, `/learn/courses/[courseSlug]`, `/learn/billing`

### Phase 3 — Learning experience
Lesson delivery, progress tracking:
`/learn/lessons/[lessonId]`

### Phase 4 — Admin operations
Core internal pages:
`/admin/courses`, `/admin/courses/new`, `/admin/courses/[courseId]`, `/admin/students`, `/admin/students/[studentId]`, `/admin/orders`, `/admin/coupons`, `/admin/settings`

### Phase 5 — Polish and expansion
Optional pages: analytics, certificates, FAQ, contact, about, pricing.

---

## 15. Error Prevention Checklist

Before generating code, verify:

- [ ] Does this table/column exist in `schema.prisma`?
- [ ] Does this route exist in `folder-architecture.md`?
- [ ] Does this feature exist in `Ecourse-platform-prd.md`?
- [ ] Does this role have permission per `permissions-matrix.md`?
- [ ] Am I enforcing permission checks server-side?
- [ ] Am I using the correct enum values from the Prisma schema?
- [ ] Am I placing this file in the correct directory per the folder architecture?
- [ ] Am I importing server code only in server contexts?
- [ ] Am I writing to the AuditLog for sensitive actions?
- [ ] Am I handling the error case, not just the happy path?

---

## 16. When You Are Unsure

If a requirement is ambiguous:
1. Preserve data integrity over convenience.
2. Prefer the simplest implementation that satisfies the PRD.
3. Ask the developer for clarification rather than guessing.
4. If you must make a decision, document your assumption with a `// TODO: Verify with PRD —` comment.

---

## 17. Seed Data Requirements

The seed script (`prisma/seed.ts`) must create:
1. All 6 roles from the `RoleCode` enum.
2. One SUPER_ADMIN user (for initial access).
3. At least one CourseCategory for testing.
4. No fake orders, enrollments, or financial data in production seeds.

---

## 18. Environment Variables

Required `.env` variables (never hardcode these):
```
DATABASE_URL=
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=
CLERK_SECRET_KEY=
CLERK_WEBHOOK_SECRET=
STRIPE_SECRET_KEY=
STRIPE_WEBHOOK_SECRET=
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=
MUX_TOKEN_ID=
MUX_TOKEN_SECRET=
MUX_WEBHOOK_SECRET=
RESEND_API_KEY=
NEXT_PUBLIC_APP_URL=
```

Never commit `.env` files. Use `.env.example` as a template with empty values.

---

## 19. Final Directive

You are building a production-grade e-course platform. Every line of code must:
- Comply with the project documents listed in Section 2.
- Respect the permission model.
- Maintain data integrity.
- Stay within MVP scope.
- Be simple, readable, and maintainable.

When in doubt, check the docs. When still in doubt, ask. Do NOT hallucinate.
