# TASK-003: ID Type Migration Plan

**Date:** 2026-09-07  
**Status:** ANALYSIS COMPLETE  
**Risk:** HIGH (touches every table, route, and type)

---

## Analysis Summary

### Current State

**Prisma Schema:**
- **15 models** use `Int @id @default(autoincrement())`
- All foreign keys also use `Int`

**Models affected:**
1. User
2. Account
3. Session
4. Resource
5. ResourceVersion
6. Purchase
7. License
8. Installation
9. Payment
10. PaymentProviderEvent
11. FinancialTransaction
12. SellerBalance (uses userId as @id)
13. Review

**Code impact:**
- **17 parseInt() calls** found in server code
- **7 route handlers** use parseInt() for domain IDs:
  - `admin.ts`: resourceId, userId, reviewId (4 uses)
  - `drm.ts`: licenseId (2 uses)
  - `payments.ts`: purchaseId (2 uses)

**Routes using :id parameter:**
- `PATCH /admin/resources/:id` → resourceId
- `PATCH /admin/users/:id/role` → userId
- `PATCH /admin/users/:id/status` → userId
- `DELETE /admin/reviews/:id` → reviewId
- `POST /purchases/:id/complete` → purchaseId
- `DELETE /licenses/:licenseId` → licenseId

**Non-domain parseInt() (safe, can stay):**
- Pagination: `page`, `limit` query params
- Config: `SMTP_PORT`, `S3_SIGNED_URL_TTL`
- Validator middleware helper

---

## Target State

### Prisma Schema Changes

**All models change to:**
```prisma
model User {
  id String @id @default(cuid())
  // ... other fields
}

model Account {
  id     String @id @default(cuid())
  userId String  // FK
  user User @relation(fields: [userId], references: [id], onDelete: Cascade)
}
```

**All foreign keys change to String:**
- `userId: Int` → `userId: String`
- `sellerId: Int` → `sellerId: String`
- `buyerId: Int` → `buyerId: String`
- `resourceId: Int` → `resourceId: String`
- `versionId: Int` → `versionId: String`
- `purchaseId: Int` → `purchaseId: String`
- `licenseId: Int` → `licenseId: String`

**JWT payload:**
```typescript
// OLD
{ userId: number }

// NEW
{ userId: string }
```

---

## Migration Strategy

### Phase 1: Schema Migration (Database)

**Option A: Create new database (RECOMMENDED for MVP)**
- Drop existing dev database
- Apply new schema
- Lose existing test data
- Clean start

**Option B: Data migration (if production data exists)**
- Create migration mapping Int → CUID
- Preserve old IDs in metadata
- NOT NEEDED YET (no production data)

**Decision:** Use **Option A** — project is pre-production, no valuable data

---

### Phase 2: TypeScript Types

**Files to update:**

1. **Generated Prisma Client** (auto-updates after schema change)
2. **JWT types** (`src/types/auth.ts` or similar)
   ```typescript
   // OLD
   interface JWTPayload {
     userId: number;
   }
   
   // NEW
   interface JWTPayload {
     userId: string;
   }
   ```

3. **Request types** (AuthRequest extends)
   ```typescript
   interface AuthRequest extends Request {
     userId?: string; // was number
   }
   ```

4. **Route parameter types** (implicit, handled by removal of parseInt)

---

### Phase 3: Route Handlers

**Remove parseInt() for domain IDs:**

**Before:**
```typescript
const userId = parseInt(req.params.id as string, 10);
```

**After:**
```typescript
const userId = req.params.id as string;
```

**Files to update:**
- `src/routes/admin.ts` (4 parseInt calls)
- `src/routes/drm.ts` (2 parseInt calls)
- `src/routes/payments.ts` (2 parseInt calls)

**Validation:** Add Zod schema to validate CUID format in middleware

---

### Phase 4: JWT Encoding/Decoding

**Files to check:**
- `src/lib/jwt.ts` or `src/middleware/auth.ts`

**Before:**
```typescript
const token = jwt.sign({ userId: user.id }, secret); // user.id is number
```

**After:**
```typescript
const token = jwt.sign({ userId: user.id }, secret); // user.id is string
```

**Compatibility:** New tokens with string userId will naturally work after schema change

---

### Phase 5: Database Queries

**Prisma queries automatically type-safe after schema change.**

**Before:**
```typescript
await prisma.user.findUnique({ where: { id: 123 } }); // number
```

**After:**
```typescript
await prisma.user.findUnique({ where: { id: 'clx123abc' } }); // string
```

TypeScript compiler will catch all mismatches.

---

## Implementation Steps

### Step 1: Backup & Branch
```bash
git checkout -b task-003-id-migration
```

### Step 2: Update Prisma Schema
- Change all `Int @id @default(autoincrement())` to `String @id @default(cuid())`
- Change all foreign key types `Int` → `String`
- Verify with `npx prisma validate`

### Step 3: Drop & Recreate Database
```bash
# Dev only - no production data exists
npx prisma migrate dev --name id_migration_to_cuid
# OR (if no migrations)
npx prisma db push --force-reset
```

### Step 4: Update TypeScript Types
- Find all `userId: number` declarations
- Change to `userId: string`
- Update JWT payload interface
- Update AuthRequest interface

### Step 5: Remove parseInt() for Domain IDs
- `src/routes/admin.ts`: 4 changes
- `src/routes/drm.ts`: 2 changes
- `src/routes/payments.ts`: 2 changes

### Step 6: Add CUID Validation Middleware
```typescript
const validateCuid = (paramName: string) => (req: Request, res: Response, next: NextFunction) => {
  const value = req.params[paramName];
  if (!/^c[a-z0-9]{24}$/i.test(value)) {
    return res.status(400).json({ error: 'Invalid ID format' });
  }
  next();
};

// Usage
router.get('/users/:id', validateCuid('id'), handler);
```

### Step 7: Run Type Check
```bash
pnpm run type-check
```
Fix all type errors revealed by TypeScript.

### Step 8: Manual Testing
- Create user via Discord OAuth
- Create resource
- Create purchase
- Activate license
- Verify all IDs are CUIDs in responses

### Step 9: Write Integration Tests
```typescript
describe('CUID ID format', () => {
  it('should return CUID for user ID', async () => {
    const res = await request(app).get('/auth/me').set('Authorization', `Bearer ${token}`);
    expect(res.body.user.id).toMatch(/^c[a-z0-9]{24}$/i);
  });
  
  it('should accept CUID in route params', async () => {
    const res = await request(app).get(`/resources/${validCuid}`);
    expect(res.status).not.toBe(400);
  });
  
  it('should reject invalid ID format', async () => {
    const res = await request(app).get('/resources/123');
    expect(res.status).toBe(400);
    expect(res.body.error).toMatch(/Invalid ID/);
  });
});
```

---

## Risks & Mitigations

### Risk 1: Breaking Changes for Frontend
**Impact:** Frontend may expect `number` IDs  
**Mitigation:** Update frontend types simultaneously, test API integration

### Risk 2: JWT Token Incompatibility
**Impact:** Existing dev tokens become invalid  
**Mitigation:** Force logout all sessions, acceptable for pre-production

### Risk 3: Third-party Integrations
**Impact:** YooKassa metadata may store Int IDs  
**Mitigation:** Check webhook payload parsing, update metadata handling

### Risk 4: Database Size
**Impact:** String IDs use more storage than Int  
**Mitigation:** CUID is 25 bytes vs 4 bytes (acceptable trade-off for security)

---

## Acceptance Criteria

- [ ] All Prisma models use `String @id @default(cuid())`
- [ ] No parseInt() calls for domain IDs remain
- [ ] TypeScript compiles without errors
- [ ] All routes accept CUID format
- [ ] Invalid ID format returns 400
- [ ] Integration tests pass with CUID IDs
- [ ] JWT tokens contain string userId
- [ ] Frontend updated to handle string IDs

---

## Estimated Effort

**Analysis:** 1 hour ✅ DONE  
**Schema changes:** 30 minutes  
**Code changes:** 2-3 hours  
**Testing:** 2 hours  
**Total:** ~5-6 hours (1 day)

---

## Next Steps

1. Create branch `task-003-id-migration`
2. Update Prisma schema
3. Run migration
4. Fix TypeScript errors
5. Add validation
6. Write tests
7. Manual verification
8. Update production-readiness.md (SEC-01: VERIFIED)

---

**Status:** READY TO IMPLEMENT  
**Blocker:** None  
**Dependencies:** None
