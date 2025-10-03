# Environment Flow Diagram

## Complete Environment Flow

```
┌────────────────────────────────────────────────────────────────────┐
│                        GITHUB REPOSITORY                           │
│                                                                    │
│  feature/xyz ──→ PR ──→ main (DEV-INT auto-deploys) ──────┐      │
│                          │                                  │      │
│                          │                                  │      │
│                 release/v1.0.0 ────────────────────────────┤      │
│                                                             │      │
└─────────────────────────────────────────────────────────────┼──────┘
                                                              │
                                                              ↓
┌────────────────────────────────────────────────────────────────────┐
│                          ENVIRONMENTS                              │
│                                                                    │
│   ┌──────────┐                          ┌──────────┐             │
│   │ DEV-INT  │ ─────────────────────→  │   PROD   │             │
│   │ (main)   │                    ┌──→ │          │             │
│   │  Auto    │                    │    └──────────┘             │
│   └──────────┘                    │         ▲                    │
│        │                          │         │                    │
│        │ deploy                   │         │ promote            │
│        │                          │         │                    │
│        ↓                          │    ┌──────────┐             │
│   ┌──────────┐    promote    ┌───────┤ STAGING  │             │
│   │  FUTURE  │ ────────────→ │ACTIVE ││          │             │
│   │          │               │       │└──────────┘             │
│   └──────────┘               └───────┘     ▲                    │
│                                   │         │                    │
│                                   │ promote │                    │
│                                   ↓         │                    │
│                              ┌──────────┐   │                    │
│                              │PARTNER/  │   │                    │
│                              │   UAT    │ ──┘                    │
│                              └──────────┘                        │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘

Legend:
  ──→  Deployment/Promotion path
  │    Connection
  ▲    Promotion direction (upward to prod)
```

## Detailed Flow with Approval Gates

```
DEPLOYMENT FLOW
═══════════════

┌─────────────┐
│   main      │  Push to main
│   branch    │ ─────────────────────────────┐
└─────────────┘                              │
                                             ↓
                                    ┌─────────────────┐
                                    │    DEV-INT      │
                                    │  (Integration)  │
                                    │   Auto-Deploy   │
                                    └─────────────────┘
                                             │
                   ┌─────────────────────────┴──────────────┐
                   │                                        │
                   ↓                                        ↓
          ┌─────────────────┐                     ┌─────────────────┐
          │     FUTURE      │                     │     ACTIVE      │
          │  (Next Version) │  ─── promote ────→ │ (Current Stable)│
          │  Manual Deploy  │                     │  Manual Deploy  │
          └─────────────────┘                     └─────────────────┘
                                                           │
                                                           │
                                  ┌────────────────────────┴─────────────────┐
                                  │                                          │
                                  ↓                                          ↓
                          ┌──────────────┐                          ┌──────────────┐
                          │ PARTNER/UAT  │                          │   STAGING    │
                          │              │   ──── promote ────→    │              │
                          └──────────────┘                          └──────────────┘
                                  │                                          │
                                  │        🔐 Approval Required             │
                                  │           (1 reviewer)                  │
                                  │                                          │
                                  └───────────────┬──────────────────────────┘
                                                  │
                                                  │ 🔒🔒 Approval Required
                                                  │    (2+ reviewers)
                                                  ↓
                                          ┌──────────────┐
                                          │     PROD     │
                                          │ (Production) │
                                          │   Live       │
                                          └──────────────┘
```

## Promotion Paths Matrix

```
FROM      │ TO: dev-int │ future │ active │  uat   │ staging │  prod  
──────────┼─────────────┼────────┼────────┼────────┼─────────┼────────
dev-int   │      ─      │   ✅   │   ✅   │   ❌   │   ❌    │   ❌
future    │      ❌     │   ─    │   ✅   │   ❌   │   ❌    │   ❌
active    │      ❌     │   ❌   │   ─    │   ✅   │   ✅    │   ❌
uat       │      ❌     │   ❌   │   ❌   │   ─    │   ✅    │   ❌
staging   │      ❌     │   ❌   │   ❌   │   ❌   │   ─     │   ✅
prod      │      ❌     │   ❌   │   ❌   │   ❌   │   ❌    │   ─

Legend:
  ✅  Valid promotion path
  ❌  Invalid promotion path
  ─   Same environment (N/A)
```

## Approval Requirements by Promotion

```
Promotion Path              │ Approvals │ Approvers
────────────────────────────┼───────────┼──────────────────────────
dev-int → future            │     0     │ None
dev-int → active            │     0     │ None (consider adding 1)
future → active             │     0     │ None (consider adding 1)
active → uat                │     1     │ QA Leads, Dev Leads
active → staging            │     1     │ Senior Devs, QA Leads
uat → staging               │     1     │ Senior Devs, QA Leads
staging → prod              │    2+     │ Senior Engineers, Managers
```

## Environment Characteristics

```
┌─────────────────────────────────────────────────────────────────────┐
│ Environment │ Stability │ Access      │ Deploy      │ Change Freq │
├─────────────┼───────────┼─────────────┼─────────────┼─────────────┤
│ DEV-INT     │ Unstable  │ Dev Team    │ Auto (main) │ Very High   │
│ FUTURE      │ Testing   │ Dev + QA    │ Manual      │ High        │
│ ACTIVE      │ Stable    │ Dev + QA    │ Manual/Prom │ Medium      │
│ UAT         │ Stable    │ QA+Partners │ Promotion   │ Medium      │
│ STAGING     │ Prod-like │ QA + Ops    │ Promotion   │ Low         │
│ PROD        │ Highest   │ Ops Team    │ Promotion   │ Very Low    │
└─────────────────────────────────────────────────────────────────────┘
```

## Typical Release Timeline

```
Day 1-5: Development & Integration
├── Feature branches → PRs → main
├── Auto-deploy to DEV-INT
└── Continuous integration testing

Day 6: Release Branch Creation
├── Create release/v2.0.0 from main
├── Deploy to FUTURE environment
└── Begin feature testing

Day 7-10: Testing Phase
├── QA testing in FUTURE
├── Bug fixes applied to release branch
├── Promote FUTURE → ACTIVE when stable
└── Extended testing in ACTIVE

Day 11-12: Partner Testing
├── Promote ACTIVE → UAT (1 approval)
├── Partners test in UAT
├── Gather feedback
└── Fix critical issues if needed

Day 13: Pre-Production
├── Promote UAT/ACTIVE → STAGING (1 approval)
├── Final validation in prod-like environment
├── Performance testing
└── Security scans

Day 14: Production Release
├── Prepare rollback plan
├── Schedule deployment window
├── Promote STAGING → PROD (2+ approvals)
├── Monitor closely
└── Success! 🎉

Post-Release: Monitoring
├── Track metrics
├── Monitor for issues
└── Be ready to rollback if needed
```

## Quick Reference Commands

### View Current Deployments
```bash
# Check what's deployed where
gh api /repos/:owner/:repo/deployments | jq '.[] | {environment, ref, created_at}'
```

### Trigger Manual Deployment
```bash
# Deploy to specific environment
gh workflow run deploy.yml -f environment=future -f release_type=future
```

### Trigger Promotion
```bash
# Promote between environments
gh workflow run promote-release.yml \
  -f release_id=release-20250103-120000 \
  -f source_environment=staging \
  -f target_environment=prod
```

### Check Workflow Status
```bash
# Monitor running workflows
gh run list --workflow=deploy.yml --limit 5
gh run list --workflow=promote-release.yml --limit 5
```

## Environment URLs (Example)

```
DEV-INT:  https://dev-int.example.com
FUTURE:   https://future.example.com
ACTIVE:   https://active.example.com
UAT:      https://uat.example.com
STAGING:  https://staging.example.com
PROD:     https://example.com (or www.example.com)
```

## Rollback Procedures

### From DEV-INT
```
Revert commit in main → Auto-redeploys
```

### From FUTURE/ACTIVE
```
Redeploy previous release ID
```

### From UAT/STAGING
```
Promote previous stable release from ACTIVE
```

### From PROD
```
Emergency rollback:
STAGING (previous) → PROD
Requires 2+ approvals (same as forward promotion)
```

## Health Check Endpoints

Each environment should expose:

```
GET /health          → Basic liveness check
GET /health/ready    → Readiness check
GET /health/metrics  → Performance metrics
GET /version         → Deployed version info
```

Example response:
```json
{
  "status": "healthy",
  "version": "v2.0.0",
  "environment": "prod",
  "commit": "abc123def456",
  "deployed_at": "2025-01-03T12:00:00Z",
  "uptime": "5d 3h 42m"
}
```

## Monitoring Dashboard Layout

```
┌─────────────────────────────────────────────────────────┐
│                  Environment Overview                   │
├──────────┬──────────┬──────────┬──────────┬─────────────┤
│ DEV-INT  │  FUTURE  │  ACTIVE  │   UAT    │   STAGING   │  PROD
│  v1.5.2  │  v2.0.0  │  v1.9.5  │  v1.9.5  │   v1.9.5    │ v1.9.4
│  🟢      │  🟢      │  🟢      │  🟢      │   🟢        │ 🟢
└──────────┴──────────┴──────────┴──────────┴─────────────┴────────┘

Recent Deployments:
  • PROD     v1.9.4  deployed  2h ago   by @alice   ✅ stable
  • STAGING  v1.9.5  deployed  1d ago   by @bob     ✅ stable
  • UAT      v1.9.5  deployed  2d ago   by @carol   ✅ stable
  • ACTIVE   v1.9.5  deployed  3d ago   by @dave    ✅ stable
  • FUTURE   v2.0.0  deployed  5d ago   by @eve     🧪 testing

Pending Approvals:
  • STAGING → PROD (v1.9.5) - Waiting for 2 approvals

Active Incidents:
  • None 🎉
```

## Best Practices Summary

✅ **DO:**
- Auto-deploy main to DEV-INT for fast feedback
- Use FUTURE for next version testing
- Keep ACTIVE stable for promotions
- Test thoroughly in UAT with partners
- Mirror production config in STAGING
- Require multiple approvals for PROD
- Monitor all environments
- Document every production deployment

❌ **DON'T:**
- Skip environments (follow the path)
- Deploy directly to PROD (use promotion)
- Promote without approvals (UAT/STAGING/PROD)
- Deploy to PROD during peak hours (plan ahead)
- Forget to test rollback procedures
- Ignore monitoring alerts

## Incident Response Flow

```
Incident Detected in PROD
        ↓
Assess Severity
        ↓
    ┌───┴────┐
    │ Minor  │         │ Critical │
    │        │         │          │
    ↓        │         ↓          │
Monitor      │    Rollback        │
Fix in       │    STAGING→PROD   │
next         │    (emergency)     │
release      │         ↓          │
             │    Investigate     │
             │    Root cause      │
             │         ↓          │
             └──→  Fix & Deploy   │
                   (expedited)    │
                        ↓         │
                   Monitor &      │
                   Document       │
```

