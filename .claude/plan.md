# WinBackk RiskMap - Implementation Plan

## Project Overview
Internal web app for WinBackk staff to conduct workplace movement risk audits and generate PDF reports. Staff-only (no client login).

## Tech Stack
- **Frontend**: Next.js 14 App Router + Tailwind CSS + shadcn/ui
- **Backend/DB**: Supabase PostgreSQL with RLS
- **Auth**: Clerk (staff-only)
- **PDF**: Playwright (server-side HTML-to-PDF)

## Key Decisions
- **MRI Algorithm**: Weighted formula (0-4 scale) from MRI ALGO.pdf
- **Delta Calculation**: Same Org + Same Site comparison
- **Scope**: MVP first (core audit flow)

---

## Phase 1: Foundation

### 1.1 Project Setup
**Files to create:**
- `package.json` - Dependencies (next, @clerk/nextjs, @supabase/supabase-js, playwright, recharts, zod, react-hook-form)
- `tailwind.config.ts` - Tailwind configuration
- `next.config.ts` - Next.js configuration
- `.env.local.example` - Environment variables template
- `tsconfig.json` - TypeScript configuration

### 1.2 Database Schema
**File**: `supabase/migrations/001_initial_schema.sql`

**Core Tables:**
```
orgs                    - Client organisations
org_sites               - Locations within orgs
org_roles               - Job roles within orgs
org_user_assignments    - Staff-to-org access mapping
exposure_drivers        - 6 weighted drivers (global reference)
audits                  - Audit records
exposure_map_rows       - Role x Driver scores (0-4)
mri_snapshots           - Calculated MRI outputs
controls                - Remedial actions
documents               - PDF reports & evidence files
audit_log               - Change tracking
```

**Exposure Drivers (seed data):**
| Name | Code | Weight |
|------|------|--------|
| Sustained Sitting | sustained_sitting | 0.25 |
| Movement Variation | movement_variation | 0.20 |
| Upper Limb Load | upper_limb_load | 0.15 |
| Neck/Shoulder Demand | neck_shoulder_demand | 0.15 |
| Work Organisation | work_organisation | 0.15 |
| Workstation Fit | workstation_fit | 0.10 |

**RLS Policies:**
- `user_has_org_access(org_id)` function checks admin role OR assignment
- All tables with `org_id` use this function for SELECT/INSERT/UPDATE
- Admin-only DELETE on most tables

### 1.3 Auth Integration
**Files:**
- `lib/clerk/auth.ts` - Auth helpers
- `middleware.ts` - Route protection
- `app/(auth)/sign-in/[[...sign-in]]/page.tsx`
- `app/(auth)/sign-up/[[...sign-up]]/page.tsx`

**User Roles:**
- `winbackk_admin` - Full access to all orgs
- `winbackk_staff` - Access only to assigned orgs

---

## Phase 2: Core CRUD (MVP)

### 2.1 Organisation Management (Minimal)
**Files:**
- `app/(dashboard)/organisations/page.tsx` - List orgs
- `app/(dashboard)/organisations/[orgId]/page.tsx` - Org details with sites/roles
- `app/api/orgs/route.ts` - CRUD endpoints
- `app/api/orgs/[orgId]/sites/route.ts`
- `app/api/orgs/[orgId]/roles/route.ts`

**Components:**
- `components/features/organisations/org-form.tsx`
- `components/features/organisations/site-manager.tsx`
- `components/features/organisations/role-manager.tsx`

### 2.2 Audit Creation
**Files:**
- `app/(dashboard)/audits/page.tsx` - List audits
- `app/(dashboard)/audits/new/page.tsx` - Create audit wizard
- `app/api/audits/route.ts`

**Wizard Steps:**
1. Select Organisation
2. Select Site
3. Pick Audit Date
4. Assign Assessor (auto-fill current user)
5. Optional Notes

### 2.3 Audit Workspace
**File**: `app/(dashboard)/audits/[auditId]/page.tsx`

**Three Tabs:**

#### Tab 1: Exposure Map
**Files:**
- `components/features/exposure-map/exposure-matrix.tsx` - Role x Driver grid
- `components/features/exposure-map/score-cell.tsx` - Score input (0-4)
- `app/api/audits/[auditId]/exposure-map/route.ts`

**Matrix Structure:**
```
           | Sitting | Movement | Upper Limb | Neck | Work Org | Workstation |
Role 1     |   [0-4] |   [0-4]  |    [0-4]   | [0-4]|   [0-4]  |    [0-4]    |
Role 2     |   [0-4] |   [0-4]  |    [0-4]   | [0-4]|   [0-4]  |    [0-4]    |
...
```

**Features:**
- Color coding: 0=green, 1=yellow, 2=orange, 3=red, 4=dark red
- Auto-save with 500ms debounce
- Confidence rating per cell (optional)
- Notes per cell (optional)

#### Tab 2: Controls
**Files:**
- `components/features/controls/controls-table.tsx`
- `components/features/controls/control-form.tsx`
- `app/api/audits/[auditId]/controls/route.ts`

**Control Fields:**
- Title, Type (elimination/substitution/engineering/administrative/PPE)
- Owner, Due Date, Priority Level
- Linked Exposure Driver
- Expected Impact, Success Metric
- Status (todo/doing/done)

#### Tab 3: Evidence
**Files:**
- `components/features/evidence/file-uploader.tsx`
- `components/features/evidence/file-list.tsx`
- `app/api/audits/[auditId]/evidence/route.ts`

**Supported Files:** jpg, jpeg, png, heic, webp, pdf, docx, xlsx, pptx, txt, md

---

## Phase 3: MRI Calculation Engine

### 3.1 Algorithm Implementation
**File**: `lib/mri/calculator.ts`

**Weighted MRI Formula:**
```typescript
MRI = 100 * SUM(weight * (score / 4)) / numRolesWithScores

// Per role:
roleScore = SUM(weight * (score / 4)) for all drivers
rolePct = roleScore * 100

// Per driver (across roles):
driverScore = AVG(score) for all roles
driverPct = (driverScore / 4) * 100
```

**Delta Calculation:**
```typescript
// Find previous audit: same org_id + same site_id + earlier audit_date
delta = current.MRI_pct - previous.MRI_pct
// Display: "+X.X points" or "-X.X points" or "N/A (first audit)"
```

### 3.2 MRI Snapshot Storage
**File**: `app/api/audits/[auditId]/calculate-mri/route.ts`

**mri_snapshots table fields:**
- `overall_score` (0-100, 1 decimal)
- `by_role_json` - Array of {role_id, role_name, raw, pct}
- `by_driver_json` - Array of {driver_id, driver_name, raw, pct}
- `top_driver_id`, `top_driver_score`
- `previous_audit_id`, `delta_points`

---

## Phase 4: PDF Report Generation

### 4.1 Report Infrastructure
**Files:**
- `lib/pdf/generator.ts` - Playwright PDF conversion
- `lib/pdf/templates/audit-report.tsx` - HTML template (React component)

### 4.2 Generation Flow
**File**: `app/api/reports/generate/route.ts`

**Steps:**
1. POST `/api/reports/generate` with {audit_id, report_type}
2. Validate user access (Clerk + RLS)
3. Fetch all audit data from Supabase
4. Calculate MRI (or use cached snapshot)
5. Render HTML template with data
6. Convert to PDF via Playwright
7. Upload to Supabase Storage: `orgs/{org_id}/audits/{audit_id}/reports/audit-report-v{N}.pdf`
8. Create `documents` row with version
9. Return signed URL (7-day expiry)

### 4.3 Report Sections
**Workplace Movement Risk Audit Report:**
1. Cover page (org, site, date, assessor)
2. Executive Summary
3. Overall MRI Score + Delta vs Previous
4. MRI by Role (table + chart)
5. MRI by Driver (table + chart)
6. Exposure Map Heatmap
7. Top Exposure Drivers
8. Controls Summary (open/done/overdue)
9. Controls Table
10. Appendix: Scoring Methodology

### 4.4 Report UI
**Files:**
- `app/(dashboard)/reports/page.tsx` - View past reports
- `app/(dashboard)/reports/generate/page.tsx` - Generate new report
- `components/features/reports/report-preview.tsx`

---

## Phase 5: Post-MVP Enhancements

### 5.1 Additional Screens (after MVP)
- Home dashboard with 6 action tiles
- Controls Register (cross-audit view)
- Documents & Evidence browser
- Audit Log viewer
- Templates & Settings (admin)

### 5.2 Control Progress Update Report
- Focused on control status changes
- Simplified template

---

## Directory Structure (MVP)

```
app/
├── (auth)/
│   ├── sign-in/[[...sign-in]]/page.tsx
│   └── sign-up/[[...sign-up]]/page.tsx
├── (dashboard)/
│   ├── layout.tsx
│   ├── page.tsx                      # Simple dashboard
│   ├── audits/
│   │   ├── page.tsx                  # List audits
│   │   ├── new/page.tsx              # Create audit
│   │   └── [auditId]/page.tsx        # Audit workspace (tabs)
│   ├── organisations/
│   │   ├── page.tsx
│   │   └── [orgId]/page.tsx
│   └── reports/
│       ├── page.tsx                  # Past reports
│       └── generate/page.tsx
├── api/
│   ├── orgs/...
│   ├── audits/...
│   └── reports/generate/route.ts
└── globals.css

components/
├── ui/                               # shadcn/ui
├── layout/
│   ├── sidebar.tsx
│   └── header.tsx
└── features/
    ├── organisations/
    ├── audits/
    ├── exposure-map/
    ├── controls/
    ├── evidence/
    ├── mri/
    └── reports/

lib/
├── supabase/
│   ├── client.ts
│   └── server.ts
├── clerk/auth.ts
├── mri/calculator.ts
├── pdf/
│   ├── generator.ts
│   └── templates/audit-report.tsx
└── utils/

supabase/
└── migrations/
    └── 001_initial_schema.sql

types/
├── database.ts
└── api.ts
```

---

## Critical Path

```
1. Database Schema + RLS
   └── 2. Clerk Integration
       └── 3. Org/Site/Role CRUD
           └── 4. Audit Creation
               └── 5. Exposure Map Scoring
                   └── 6. MRI Calculator
                       └── 7. PDF Generation
```

---

## Reference Documents

| Document | Purpose |
|----------|---------|
| `Documents/Movement Risk Index ALGO.pdf` | MRI algorithm with 6 weighted domains (0-4 scale) |
| `Documents/WinBackk Client Portal.pdf` | Developer handover: tables, calculations, PDF flow |
| `Documents/WinBackk RiskMap (2).pdf` | UI wireframes for all screens |

---

## Acceptance Criteria

- [ ] Staff can create orgs, sites, roles
- [ ] Staff can create and continue audits
- [ ] Exposure map allows 0-4 scoring per role/driver
- [ ] MRI calculated using weighted formula
- [ ] Delta shows comparison to previous audit (same org+site)
- [ ] "Generate PDF" produces professional report
- [ ] PDF stored privately with signed download URL
- [ ] RLS prevents cross-org data access
- [ ] Audit log tracks score/control changes
