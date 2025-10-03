# Environment Flow Documentation

This document describes the environment promotion flow and deployment strategy.

## Environment Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                         ENVIRONMENTS                            │
│                                                                 │
│                             PROD                                │
│                              ▲                                  │
│                              │                                  │
│   FUTURE  →  ACTIVE  →  STAGING                                │
│                              ▲                                  │
│                              │                                  │
│   DEV-INT  ────────────→  PARTNER/UAT                          │
│    (main)                                                       │
└─────────────────────────────────────────────────────────────────┘
```

## Environment Descriptions

### 1. DEV-INT (Development Integration)
- **Source**: `main` branch
- **Deployment**: Automatic on push to main
- **Purpose**: Integration environment for merged features
- **Stability**: Unstable - frequent changes
- **Access**: Development team
- **Promotion Path**: Can promote to FUTURE or ACTIVE

### 2. FUTURE
- **Source**: Next version releases
- **Deployment**: Manual deployment for upcoming releases
- **Purpose**: Preview/test future releases before they go active
- **Stability**: Testing new features
- **Access**: Development + QA teams
- **Promotion Path**: Promotes to ACTIVE

### 3. ACTIVE
- **Source**: Current active release version
- **Deployment**: Promoted from FUTURE or deployed directly
- **Purpose**: Currently active/stable release
- **Stability**: Stable - tested version
- **Access**: Development + QA teams
- **Promotion Path**: Can promote to UAT or STAGING

### 4. PARTNER/UAT (User Acceptance Testing)
- **Source**: Promoted from ACTIVE
- **Deployment**: Manual promotion with approval
- **Purpose**: Partner/customer testing and validation
- **Stability**: Stable - pre-production testing
- **Access**: QA team + Partners/Customers
- **Promotion Path**: Promotes to STAGING

### 5. STAGING
- **Source**: Promoted from ACTIVE or UAT
- **Deployment**: Manual promotion with approval
- **Purpose**: Final pre-production validation
- **Stability**: Production-like - final testing
- **Access**: QA + Operations teams
- **Promotion Path**: Promotes to PROD
- **Note**: Should mirror production configuration

### 6. PROD (Production)
- **Source**: Promoted from STAGING only
- **Deployment**: Manual promotion with strict approval (2+ reviewers)
- **Purpose**: Live production environment
- **Stability**: Highest - serving customers
- **Access**: Operations team + Senior engineers
- **Promotion Path**: None (final destination)

## Valid Promotion Paths

### Primary Flow
```
DEV-INT → FUTURE → ACTIVE → STAGING → PROD
  (main)                        ▲         ▲
                                │         │
                              UAT ────────┘
```

### Detailed Paths

1. **DEV-INT → FUTURE**
   - Deploy next version to FUTURE for testing
   - Manual deployment
   - No approval required

2. **DEV-INT → ACTIVE**
   - Direct deployment of main to ACTIVE
   - Manual deployment
   - No approval required (but consider adding for safety)

3. **FUTURE → ACTIVE**
   - Promote tested future release to active
   - Manual promotion
   - Optional approval

4. **ACTIVE → UAT**
   - Send active release to partner/UAT testing
   - Manual promotion
   - Approval recommended

5. **ACTIVE → STAGING**
   - Direct path to staging (skip UAT if not needed)
   - Manual promotion
   - Approval required

6. **UAT → STAGING**
   - After partner validation, promote to staging
   - Manual promotion
   - Approval required

7. **STAGING → PROD**
   - Final promotion to production
   - Manual promotion
   - **2+ approvals required**
   - Strictest validation

## Deployment Types

### Automatic Deployment
- **Trigger**: Push to `main` branch
- **Target**: DEV-INT only
- **Process**:
  1. Developer merges PR to main
  2. Workflow automatically triggers
  3. Deploys to DEV-INT immediately
  4. No approval needed

### Manual Deployment
- **Trigger**: Workflow dispatch
- **Target**: FUTURE, ACTIVE, UAT, STAGING, PROD
- **Process**:
  1. Developer triggers workflow manually
  2. Selects target environment
  3. Validation checks run
  4. Branch validation ensures correct source branch
  5. Deploys to selected environment

### Promotion Workflow
- **Trigger**: Workflow dispatch (promote-release.yml)
- **Target**: Any valid promotion path
- **Process**:
  1. Developer triggers promotion workflow
  2. Specifies release ID, source, and target environments
  3. Promotion path validation
  4. **Approval gate** (for UAT → STAGING → PROD)
  5. Pre-promotion checks
  6. Execute promotion
  7. Post-promotion verification

## Branch Strategy

### Main Branch
- **Environment**: DEV-INT
- **Protection**: 
  - Require PR reviews
  - Require status checks
  - No direct pushes (except admins)
- **Deployment**: Auto-deploy to DEV-INT on merge

### Release Branches (`release/*`)
- **Environment**: FUTURE, ACTIVE, UAT, STAGING, PROD
- **Protection**:
  - Require PR reviews
  - Require status checks
  - No force pushes
- **Deployment**: Manual deployment to any environment
- **Naming**: `release/v1.2.3` or `release/2025-01-03`

### Feature Branches (`feature/*`)
- **Environment**: None (not directly deployed)
- **Flow**: Merged to main → auto-deployed to DEV-INT

## Approval Requirements

### No Approval Required
- ✅ Automatic deployment to DEV-INT
- ✅ Manual deployment to DEV-INT, FUTURE, ACTIVE

### Single Approval Required
- 🔐 Promotion to UAT
- 🔐 Promotion to STAGING (from ACTIVE or UAT)
- **Reviewers**: Development leads, QA leads

### Multiple Approvals Required
- 🔒🔒 Promotion to PROD
- **Reviewers**: Senior engineers, Engineering managers, Operations leads
- **Minimum**: 2 reviewers
- **Recommended**: 3 reviewers for critical releases

## Configuration Required

### GitHub Environments to Create

1. **dev-int**
   - No protection rules
   - Auto-deploy on main push

2. **future**
   - No protection rules (or optional 1 reviewer)
   - Manual deployment only

3. **active**
   - Optional 1 reviewer for extra safety
   - Manual deployment only

4. **promotion-approval-uat**
   - Required reviewers: 1
   - Reviewers: QA leads, Development leads

5. **uat**
   - Deploy after approval gate passes
   - Environment-specific secrets

6. **promotion-approval-staging**
   - Required reviewers: 1
   - Reviewers: Senior developers, QA leads

7. **staging**
   - Deploy after approval gate passes
   - Should mirror production config
   - Environment-specific secrets

8. **promotion-approval-prod**
   - Required reviewers: 2+
   - Reviewers: Senior engineers, Engineering managers, SRE team
   - Optional wait timer: 15-30 minutes

9. **prod**
   - Deployment branches: `main`, `release/*`
   - Production secrets
   - Highest protection level

## Example Workflows

### Scenario 1: New Feature Development
```
1. Developer creates feature branch
2. Developer opens PR to main
3. PR reviewed and approved
4. PR merged to main
5. ✅ Auto-deploys to DEV-INT
6. Test in DEV-INT
```

### Scenario 2: Release Preparation
```
1. Create release branch: release/v2.0.0
2. Manually deploy to FUTURE environment
3. QA tests in FUTURE
4. When stable, promote FUTURE → ACTIVE
5. Continue testing in ACTIVE
```

### Scenario 3: Partner Testing
```
1. Release is stable in ACTIVE
2. Trigger promotion: ACTIVE → UAT
3. Validation checks run
4. Approval gate (1 reviewer approves)
5. Deploy to UAT
6. Partners test in UAT environment
7. Gather feedback
```

### Scenario 4: Production Deployment
```
1. Release fully tested in ACTIVE/UAT
2. Trigger promotion: STAGING → PROD
3. Validation checks run
4. Pre-promotion checks (security, health, compliance)
5. Approval gate (2+ reviewers must approve)
6. Optional wait timer (30 min for production)
7. Execute promotion to PROD
8. Post-promotion verification
9. Monitor for issues
10. ✅ Production deployment complete
```

### Scenario 5: Hotfix
```
1. Create hotfix branch from main or release
2. Apply fix
3. Deploy to DEV-INT for quick test
4. Promote to ACTIVE for validation
5. If urgent, promote ACTIVE → STAGING → PROD
6. Standard approvals still required for PROD
```

## Rollback Strategy

### Rollback from DEV-INT
- Simply redeploy previous commit from main
- Or revert commit in main (triggers auto-deploy)

### Rollback from FUTURE/ACTIVE
- Redeploy previous release ID
- Or deploy from earlier commit/tag

### Rollback from UAT/STAGING
- Promote previous stable release from ACTIVE
- Or redeploy previous release ID

### Rollback from PROD
- Promote previous release from STAGING
- Or use dedicated rollback workflow (create if needed)
- Requires same approval level as promotion
- Document rollback reason for audit

## Monitoring and Validation

### Post-Deployment Checks
Each environment should have:
- ✅ Health check endpoints
- ✅ Smoke tests
- ✅ Critical path validation
- ✅ Performance monitoring
- ✅ Error rate tracking
- ✅ Alerting configured

### Environment-Specific Monitoring

**DEV-INT**: Basic health checks, log errors  
**FUTURE/ACTIVE**: Health + basic metrics  
**UAT**: Health + user journey monitoring  
**STAGING**: Full production-like monitoring  
**PROD**: Comprehensive monitoring + alerting + on-call

## Best Practices

1. **Always test in lower environments first**
   - Never skip environments
   - Each environment serves a specific purpose

2. **Maintain environment parity**
   - STAGING should closely mirror PROD
   - Use same infrastructure, configs (with different values)

3. **Document releases**
   - Use release notes
   - Track what's deployed where
   - Maintain changelog

4. **Automate where possible**
   - Auto-deploy to DEV-INT ✅
   - Automated checks and validations ✅
   - Manual approvals for critical environments ✅

5. **Monitor everything**
   - Track deployment metrics
   - Monitor application health
   - Alert on anomalies

6. **Have a rollback plan**
   - Test rollback procedures
   - Keep previous versions available
   - Document rollback process

7. **Review approval history**
   - Audit who approved what
   - Ensure compliance
   - Learn from incidents

## Troubleshooting

### Deployment stuck in DEV-INT
- Check GitHub Actions logs
- Verify main branch is healthy
- Check if workflow is disabled

### Promotion validation fails
- Review error message
- Check promotion path is valid
- Verify source release exists and is healthy

### Approval not working
- Verify reviewers have correct permissions
- Check environment protection rules
- Ensure reviewers are notified

### Environment health check fails
- Review application logs
- Check infrastructure status
- Verify configuration is correct
- Rollback if necessary

## Additional Resources

- [Main README](../README.md)
- [Approval Setup Guide](SETUP_APPROVALS.md)
- [GitHub Environments Docs](https://docs.github.com/en/actions/deployment/targeting-different-environments)

