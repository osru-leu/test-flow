# Deployment Flow Workflow

This repository contains a modular GitHub Actions workflow that implements the deployment flow diagram with proper validation gates and failure handling.

## Flow Overview

### Initial Deployment Flow

```
GitHub Actions (Deployment Trigger)
           ↓
Deployment Lifecycle Validation
    ↓ (Yes)          ↓ (No)
Branch Validation    Workflow Stops (Failure)
    ↓
Future or Active Release Check
    ↓ (Yes)          ↓ (Check fails)
Deploy to Future Release    Workflow Stops (Failure)
(Duration - Unlimited)
```

### Environment Structure

```
┌──────────────────────────────────────────────────────────────┐
│                      ENVIRONMENTS                            │
│                                                              │
│                          PROD                                │
│                           ▲                                  │
│                           │                                  │
│  FUTURE → ACTIVE →   STAGING                                │
│                           ▲                                  │
│                           │                                  │
│  DEV-INT ───────────→  PARTNER/UAT                          │
│  (main)                                                      │
└──────────────────────────────────────────────────────────────┘
```

**Valid Promotion Paths:**
- DEV-INT → FUTURE or ACTIVE
- FUTURE → ACTIVE
- ACTIVE → UAT or STAGING
- UAT → STAGING
- STAGING → PROD

### Promotion Flow (with Approval Gates)

```
Promotion Request
       ↓
Validate Promotion Eligibility
       ↓ (Valid)
🔐 Approval Gate (Manual Review)
       ↓ (Approved)
Pre-Promotion Checks
       ↓ (Passed)
Execute Promotion
       ↓
Post-Promotion Verification
       ↓
✅ Complete
```

## Workflow Structure

### 1. Main Deployment Workflow
- **File**: `.github/workflows/deploy.yml`
- **Trigger**: Manual dispatch with environment and release type selection
- **Jobs**:
  1. Deployment Lifecycle Validation
  2. Branch Validation
  3. Future or Active Release Check
  4. Deploy to Future Release

### 2. Promotion Workflow (with Approvals)
- **File**: `.github/workflows/promote-release.yml`
- **Trigger**: Manual dispatch to promote existing release
- **Jobs**:
  1. Validate Promotion Request
  2. **Approval Gate** (requires authorized reviewer)
  3. Pre-Promotion Checks
  4. Execute Promotion
  5. Post-Promotion Verification

### Modular Actions

#### 1. Deployment Lifecycle Validation
- **Path**: `.github/actions/deployment-lifecycle-validation/`
- **Purpose**: Validates deployment lifecycle requirements
- **Checks**:
  - Deployment timing/windows
  - Prerequisites completion
  - Previous deployment status

#### 2. Branch Validation
- **Path**: `.github/actions/branch-validation/`
- **Purpose**: Validates branch meets deployment requirements
- **Checks**:
  - Branch naming conventions
  - Environment-specific branch rules
  - Required approvals
  - CI/CD status

#### 3. Release Check
- **Path**: `.github/actions/release-check/`
- **Purpose**: Verifies future or active release availability
- **Checks**:
  - Release window availability
  - Release schedule validation
  - Release capacity

#### 4. Deploy to Release
- **Path**: `.github/actions/deploy-to-release/`
- **Purpose**: Executes the actual deployment
- **Actions**:
  - Deploys application
  - Runs post-deployment checks
  - Configures unlimited duration deployment

### Promotion Actions

#### 5. Promotion Validation
- **Path**: `.github/actions/promotion-validation/`
- **Purpose**: Validates if a release can be promoted
- **Checks**:
  - Valid promotion path (dev → staging → production)
  - Release exists in source environment
  - Minimum soak time met
  - No blocking issues

#### 6. Pre-Promotion Checks
- **Path**: `.github/actions/pre-promotion-checks/`
- **Purpose**: Comprehensive checks before promotion
- **Checks**:
  - Health in source environment
  - Test coverage and results
  - Security scans
  - Compliance requirements
  - Rollback plan availability
  - Target environment readiness
  - Dependencies verification

#### 7. Execute Promotion
- **Path**: `.github/actions/execute-promotion/`
- **Purpose**: Executes the actual promotion
- **Actions**:
  - Retrieves artifacts from source
  - Updates configuration for target
  - Deploys to target environment
  - Runs smoke tests

#### 8. Post-Promotion Verification
- **Path**: `.github/actions/post-promotion-verification/`
- **Purpose**: Verifies promotion success
- **Checks**:
  - Application health
  - Critical functionality
  - Performance metrics
  - Integration points
  - Error rates

## Usage

### Auto-Deploy to DEV-INT

**Automatic deployment when pushing to main:**

```bash
git checkout main
git pull
git merge feature/my-feature
git push origin main
# ✅ Automatically deploys to DEV-INT
```

### Trigger Manual Deployment

1. Go to Actions tab in GitHub
2. Select "Deployment Workflow"
3. Click "Run workflow"
4. Select:
   - **Environment**: dev-int, future, active, uat, staging, or prod
   - **Release Type**: future, active, or standard (optional)
5. Click "Run workflow"

### Example Deployments

#### Deploy to FUTURE Environment

```yaml
# Triggered manually with inputs:
environment: future
release_type: future
```

Use this to deploy next version for testing.

#### Deploy to ACTIVE Environment

```yaml
# Triggered manually with inputs:
environment: active
release_type: active
```

Deploy current stable release.

#### Deploy to UAT

```yaml
# Triggered manually with inputs:
environment: uat
release_type: standard
```

Direct deployment to UAT (or use promotion workflow from ACTIVE).

### Promote Release Between Environments

1. Go to Actions tab in GitHub
2. Select "Promote Release Between Environments"
3. Click "Run workflow"
4. Provide:
   - **Release ID**: ID of the release to promote (from source environment)
   - **Source Environment**: dev or staging
   - **Target Environment**: staging or production
5. Click "Run workflow"
6. **Wait for approval** from authorized reviewers
7. Promotion executes after approval

### Example Promotions

#### Promote FUTURE → ACTIVE

```yaml
# Triggered manually with inputs:
release_id: release-20250103-120000
source_environment: future
target_environment: active
```

No approval required (optional).

#### Promote ACTIVE → UAT

```yaml
release_id: release-20250103-120000
source_environment: active
target_environment: uat
```

Requires 1 approval.

#### Promote STAGING → PROD

```yaml
release_id: release-20250103-120000
source_environment: staging
target_environment: prod
```

**Requires 2+ approvals** - most critical promotion.

**Approval Process**:
1. Promotion request is validated
2. Workflow pauses at approval gate
3. Authorized reviewers receive notification
4. Reviewers can approve/reject in GitHub UI
5. After approval, promotion proceeds automatically

**Valid Promotion Paths**:
- ✅ dev-int → future or active
- ✅ future → active
- ✅ active → uat or staging
- ✅ uat → staging
- ✅ staging → prod
- ❌ dev-int → prod (must go through intermediate environments)
- ❌ Demotion (backwards promotion)

## Setting Up Approval Gates

To enable manual approvals for promotions:

### 1. Create GitHub Environments

In your repository settings, create these environments:

#### For Approval Gates:
- `promotion-approval-uat` - 1 reviewer (QA/Dev leads)
- `promotion-approval-staging` - 1 reviewer (Senior devs/QA leads)
- `promotion-approval-prod` - 2+ reviewers (Senior engineers/Managers)

#### For Deployment Targets:
- `dev-int` - Auto-deploy from main, no protection
- `future` - Manual deploy, no protection
- `active` - Manual deploy, optional protection
- `uat` - Deploy after approval
- `staging` - Deploy after approval
- `prod` - Deploy after approval, strictest protection

### 2. Environment Details

| Environment | Auto-Deploy | Branch | Approvals | Purpose |
|------------|-------------|---------|-----------|---------|
| **dev-int** | ✅ Yes (main) | main | None | Integration testing |
| **future** | Manual | main, release/* | None | Next version testing |
| **active** | Manual | main, release/* | Optional | Current stable release |
| **uat** | Promotion | - | 1 | Partner/UAT testing |
| **staging** | Promotion | - | 1 | Pre-prod validation |
| **prod** | Promotion | - | 2+ | Production |

### 3. How Approvals Work

When a promotion workflow runs:

1. **Validation Stage**: Automatic checks run first
2. **Approval Pause**: Workflow pauses and notifies reviewers
3. **Review**: Reviewers see:
   - Release ID being promoted
   - Source and target environments
   - Who requested the promotion
   - Full workflow details
4. **Decision**: Reviewers can:
   - ✅ Approve: Promotion continues
   - ❌ Reject: Workflow fails, promotion blocked
5. **Execution**: After approval, remaining steps run automatically

## Customization

Each action contains TODO comments indicating where to implement your specific logic:

### Deployment Lifecycle Validation
Edit `.github/actions/deployment-lifecycle-validation/action.yml`:
```bash
# Add your validation logic:
# - Check deployment windows
# - Verify prerequisites
# - Validate approval status
```

### Branch Validation
Edit `.github/actions/branch-validation/action.yml`:
```bash
# Customize branch rules per environment:
if [[ "${environment}" == "production" ]]; then
  # Add production-specific rules
fi
```

### Release Check
Edit `.github/actions/release-check/action.yml`:
```bash
# Integrate with your release management system:
# - API calls to release system
# - Release schedule validation
# - Capacity checks
```

### Deploy to Release
Edit `.github/actions/deploy-to-release/action.yml`:
```bash
# Add your deployment steps:
# - Kubernetes deployment
# - Cloud platform deployment
# - Database migrations
# - Smoke tests
```

## Failure Handling

The workflow implements proper failure handling as shown in the diagram:

1. **Lifecycle Validation Failure**: Workflow stops immediately
2. **Branch Validation Failure**: Workflow stops, previous job outputs preserved
3. **Release Check Failure**: Workflow stops, deployment is prevented

All failures are logged with clear error messages and visible in GitHub Actions UI.

## Features

### Deployment Features
- ✅ Modular and reusable actions
- ✅ Environment-specific validation rules
- ✅ Proper failure handling at each gate
- ✅ Clear logging and error messages
- ✅ Manual trigger with configurable inputs
- ✅ Job dependency management
- ✅ Output propagation between jobs
- ✅ Deployment summaries

### Promotion Features
- ✅ **Manual approval gates** for controlled promotions
- ✅ **Multi-stage validation** before promotion
- ✅ **Environment protection** using GitHub environments
- ✅ **Promotion path validation** (enforces dev → staging → production)
- ✅ **Pre-promotion checks** (health, tests, security, compliance)
- ✅ **Post-promotion verification** (health, metrics, functionality)
- ✅ **Audit trail** of all approvals and promotions
- ✅ **Rollback-ready** with verification steps

## Quick Start Guide

### 1. Initial Setup
```bash
# Create GitHub environments in repository settings
- dev-int (no protection)
- future (no protection)  
- active (optional protection)
- promotion-approval-uat (1 reviewer)
- uat (environment secrets)
- promotion-approval-staging (1 reviewer)
- staging (environment secrets)
- promotion-approval-prod (2+ reviewers)
- prod (strict protection + secrets)
```

### 2. Daily Development Flow
```bash
# 1. Develop feature
git checkout -b feature/new-feature
# ... make changes ...
git commit -m "Add new feature"

# 2. Create PR to main
gh pr create --base main

# 3. After approval, merge
gh pr merge

# 4. ✅ Auto-deploys to DEV-INT
# Monitor deployment in Actions tab
```

### 3. Release Flow
```bash
# 1. Create release branch
git checkout -b release/v2.0.0

# 2. Deploy to FUTURE for testing
# Trigger workflow: environment=future

# 3. Promote to ACTIVE when stable
# Trigger promotion: future → active

# 4. Promote to UAT for partner testing
# Trigger promotion: active → uat (requires 1 approval)

# 5. Promote to STAGING
# Trigger promotion: uat → staging (requires 1 approval)

# 6. Promote to PROD
# Trigger promotion: staging → prod (requires 2+ approvals)
```

## Next Steps

1. ✅ Configure GitHub environments (see Setup Guide above)
2. ✅ Add environment-specific secrets and variables
3. 📝 Implement actual deployment logic in actions (TODOs marked)
4. 📝 Integrate with your infrastructure (K8s, cloud provider, etc.)
5. 📝 Add monitoring and alerting
6. 📝 Test the full flow in non-production environments
7. 📝 Document your specific deployment procedures
8. 📝 Set up Slack/Teams notifications (optional)

## Additional Documentation

- 📖 [Environment Flow Details](docs/ENVIRONMENT_FLOW.md) - Comprehensive environment guide
- 📖 [Approval Setup Guide](docs/SETUP_APPROVALS.md) - Detailed approval configuration

## Directory Structure

```
.github/
├── workflows/
│   ├── deploy.yml                          # Main deployment workflow
│   └── promote-release.yml                 # Promotion workflow (with approvals)
└── actions/
    ├── deployment-lifecycle-validation/
    │   └── action.yml                      # Lifecycle validation
    ├── branch-validation/
    │   └── action.yml                      # Branch validation
    ├── release-check/
    │   └── action.yml                      # Release availability check
    ├── deploy-to-release/
    │   └── action.yml                      # Deployment execution
    ├── promotion-validation/
    │   └── action.yml                      # Promotion eligibility check
    ├── pre-promotion-checks/
    │   └── action.yml                      # Comprehensive pre-promotion checks
    ├── execute-promotion/
    │   └── action.yml                      # Execute promotion between environments
    └── post-promotion-verification/
        └── action.yml                      # Post-promotion verification
```

## Contributing

When extending this workflow:
1. Keep actions modular and single-purpose
2. Add proper error handling
3. Include clear logging
4. Update this README with new features
5. Test in non-production environments first
