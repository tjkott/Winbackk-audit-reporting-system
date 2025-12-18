# WinBackk RiskMap - Orchestrator Instructions

You are the orchestrator for **WinBackk RiskMap**, an internal web application for conducting workplace movement risk audits and generating professional PDF reports.

## Project Overview

**Purpose**: Enable WinBackk staff (NOT clients) to:
- Manage client organisations, sites, and job roles
- Conduct Movement Risk Index (MRI) audits
- Score exposure drivers using a 0-4 scale
- Generate professional PDF audit reports
- Track remedial controls and evidence

**Tech Stack**:
- **Frontend**: Next.js 14 App Router + Tailwind CSS + shadcn/ui
- **Backend/DB**: Supabase PostgreSQL with Row-Level Security (RLS)
- **Auth**: Clerk (staff-only, no client access)
- **PDF**: Playwright (server-side HTML-to-PDF)

## Implementation Plan Reference

The approved plan is at: `/home/theja/.claude/plans/zany-nibbling-lollipop.md`
Also available locally at: `.claude/plan.md`

## Critical Project Specifications

### MRI Algorithm (Weighted Formula)
```typescript
// Exposure Drivers with weights (MUST total 1.0):
const DRIVERS = [
  { code: 'sustained_sitting', weight: 0.25 },
  { code: 'movement_variation', weight: 0.20 },
  { code: 'upper_limb_load', weight: 0.15 },
  { code: 'neck_shoulder_demand', weight: 0.15 },
  { code: 'work_organisation', weight: 0.15 },
  { code: 'workstation_fit', weight: 0.10 },
];

// Score scale: 0-4 (NOT 0-5)
// MRI Formula:
MRI = 100 * SUM(weight * (score / 4)) // per role
overallMRI = AVG(roleMRI) // across all roles
```

### Delta Calculation
- Compare current audit to previous audit with SAME org_id + SAME site_id
- Display: "+X.X points" / "-X.X points" / "N/A (first audit)"

### Multi-Tenant Security
- Every table with user data has `org_id` column
- RLS policy: `user_has_org_access(org_id)` function
- Admin role: `winbackk_admin` (full access)
- Staff role: `winbackk_staff` (assigned orgs only)

---

## Your Role as Orchestrator

You delegate specific tasks to specialized agents while maintaining overall project context. Your responsibilities:

1. **Track Progress**: Use TodoWrite to maintain task lists
2. **Delegate Wisely**: Assign specific, focused tasks to appropriate agents
3. **Preserve Context**: Keep tasks small enough that agents can complete them
4. **Quality Control**: Review agent outputs before proceeding
5. **Handle Blockers**: Escalate to stuck agent when issues arise

---

## Available Agents

### supabase-builder
**Purpose**: Database schema, RLS policies, and Supabase configuration
**Use When**: Creating/modifying database schema, setting up RLS, seed data
**Output**: SQL migrations in `supabase/migrations/`

### nextjs-builder
**Purpose**: Next.js frontend with Clerk auth and Supabase integration
**Use When**: Creating pages, components, API routes, layouts
**Output**: Files in `app/`, `components/`, `lib/`

### mri-calculator
**Purpose**: MRI algorithm implementation and calculation logic
**Use When**: Implementing the weighted MRI formula, delta calculations
**Output**: `lib/mri/calculator.ts` and related functions

### pdf-generator
**Purpose**: Playwright PDF generation from HTML templates
**Use When**: Creating PDF report templates and generation logic
**Output**: `lib/pdf/` directory, report templates

### coder
**Purpose**: General implementation for specific todo items
**Use When**: Focused coding tasks that don't fit other agents
**Output**: Varies by task

### tester
**Purpose**: Visual testing using Playwright MCP
**Use When**: After implementation to verify UI renders correctly
**Output**: Test reports with screenshots

### stuck
**Purpose**: Human escalation for errors or blockers
**Use When**: ANY agent encounters errors, uncertainty, or blockers
**Output**: User decision and next steps

---

## Implementation Phases

### Phase 1: Foundation
1. **Project Setup**
   - Initialize Next.js project with dependencies
   - Configure Tailwind CSS + shadcn/ui
   - Set up environment variables

2. **Database Schema** (use supabase-builder)
   - Create all 11 tables with proper relationships
   - Set up RLS policies for multi-tenant security
   - Seed exposure_drivers table with 6 weighted drivers

3. **Auth Integration** (use nextjs-builder)
   - Configure Clerk middleware
   - Create sign-in/sign-up pages
   - Set up protected routes

### Phase 2: Core CRUD
4. **Organisation Management** (use nextjs-builder + coder)
   - Org list page with create/edit
   - Site manager (inline on org page)
   - Role manager (inline on org page)

5. **Audit Flow** (use nextjs-builder + coder)
   - Audit list page
   - Create audit wizard (5 steps)
   - Audit workspace with 3 tabs

6. **Audit Workspace Tabs** (use coder for each)
   - Tab 1: Exposure Map (role x driver matrix)
   - Tab 2: Controls (remedial actions table)
   - Tab 3: Evidence (file uploads)

### Phase 3: MRI Calculation
7. **MRI Engine** (use mri-calculator)
   - Implement weighted formula
   - Calculate per-role and per-driver scores
   - Delta calculation logic
   - Snapshot storage

### Phase 4: PDF Reports
8. **PDF Generation** (use pdf-generator)
   - HTML template for audit report
   - Playwright PDF conversion
   - Supabase Storage upload
   - Document versioning

---

## Orchestration Rules

### When Delegating Tasks

**DO:**
- Give agents ONE focused task at a time
- Include all necessary context (file paths, requirements)
- Reference the plan for specifications
- Check agent output before proceeding

**DON'T:**
- Assign multiple unrelated tasks to one agent
- Skip context that agents need
- Proceed without verifying completion
- Let agents make architectural decisions

### Task Sizing

**Good task sizes:**
- "Create the database schema SQL migration"
- "Build the org list page with CRUD operations"
- "Implement the MRI calculator function"
- "Create the PDF report HTML template"

**Bad task sizes:**
- "Build the entire backend" (too large)
- "Fix typo on line 42" (too small, just do it yourself)
- "Create everything for Phase 2" (multiple tasks)

### Error Handling

When ANY agent encounters an error:
1. Agent MUST invoke stuck agent immediately
2. Stuck agent asks user for guidance
3. User decision is relayed back
4. Work continues based on user direction

---

## File Structure Reference

```
app/
├── (auth)/
│   ├── sign-in/[[...sign-in]]/page.tsx
│   └── sign-up/[[...sign-up]]/page.tsx
├── (dashboard)/
│   ├── layout.tsx
│   ├── page.tsx
│   ├── audits/
│   │   ├── page.tsx
│   │   ├── new/page.tsx
│   │   └── [auditId]/page.tsx
│   ├── organisations/
│   │   ├── page.tsx
│   │   └── [orgId]/page.tsx
│   └── reports/
│       └── page.tsx
├── api/
│   ├── orgs/...
│   ├── audits/...
│   └── reports/generate/route.ts
└── globals.css

components/
├── ui/                    # shadcn/ui components
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

## Database Tables (Quick Reference)

| Table | Purpose | Key Fields |
|-------|---------|------------|
| orgs | Client organisations | name, industry, created_by |
| org_sites | Locations within orgs | org_id, name, address |
| org_roles | Job roles within orgs | org_id, name, description |
| org_user_assignments | Staff-to-org mapping | org_id, user_id, assigned_by |
| exposure_drivers | 6 weighted drivers (global) | code, name, weight |
| audits | Audit records | org_id, site_id, audit_date, status |
| exposure_map_rows | Role x Driver scores | audit_id, role_id, driver_id, score (0-4) |
| mri_snapshots | Calculated MRI outputs | audit_id, overall_score, by_role_json |
| controls | Remedial actions | audit_id, title, type, status |
| documents | PDF reports & files | audit_id, storage_path, version |
| audit_log | Change tracking | table_name, record_id, action, changes_json |

---

## Quick Start Commands

```bash
# Start development
npm run dev

# Run Supabase locally
npx supabase start

# Apply migrations
npx supabase db push

# Generate types from Supabase
npx supabase gen types typescript --local > types/database.ts
```

---

## Quality Checklist

Before marking any phase complete:

- [ ] All pages render without errors
- [ ] RLS policies block unauthorized access
- [ ] Forms validate input correctly
- [ ] Auto-save works with debounce
- [ ] Color coding matches score values
- [ ] MRI calculation is accurate
- [ ] PDF generates with all sections
- [ ] No TypeScript errors
- [ ] Responsive on mobile/tablet/desktop

---

**You are the orchestrator. Delegate wisely, track progress, and deliver a complete audit management system.**
