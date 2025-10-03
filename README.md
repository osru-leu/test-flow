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

### Trigger Deployment Manually

1. Go to Actions tab in GitHub
2. Select "Deployment Workflow"
3. Click "Run workflow"
4. Select:
   - **Environment**: dev, staging, or production
   - **Release Type**: future or active
5. Click "Run workflow"

### Example: Production Deployment

```yaml
# Triggered manually with inputs:
environment: production
release_type: future
```

Requirements for production:
- Must be from `main`, `master`, or `release/*` branch
- Must pass lifecycle validation
- Must have valid future/active release

### Example: Development Deployment

```yaml
# Triggered manually with inputs:
environment: dev
release_type: active
```

More lenient requirements for development environment.

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

### Example: Staging to Production Promotion

```yaml
# Triggered manually with inputs:
release_id: release-20250103-120000
source_environment: staging
target_environment: production
```

**Approval Process**:
1. Promotion request is validated
2. Workflow pauses at approval gate
3. Authorized reviewers receive notification
4. Reviewers can approve/reject in GitHub UI
5. After approval, promotion proceeds automatically

**Valid Promotion Paths**:
- ✅ dev → staging
- ✅ staging → production
- ❌ dev → production (must go through staging)
- ❌ Demotion (staging → dev)

## Setting Up Approval Gates

To enable manual approvals for promotions:

### 1. Create GitHub Environments

In your repository settings, create these environments:

#### Environment: `promotion-approval-staging`
- **Reviewers**: Add team members who can approve staging promotions
- **Wait timer**: 0 minutes (optional)
- **Deployment branches**: All branches

#### Environment: `promotion-approval-production`
- **Reviewers**: Add senior team members/managers for production approvals
- **Required reviewers**: 2 (recommended for production)
- **Wait timer**: 0 minutes (or add delay if needed)
- **Deployment branches**: All branches

### 2. Configure Environment-Specific Settings

You can also add environment protection for the actual deployment environments:

#### Environment: `production`
- **Reviewers**: Production deployment approvers
- **Environment secrets**: Add production credentials
- **Environment variables**: Production-specific configs

#### Environment: `staging`
- **Environment secrets**: Staging credentials
- **Environment variables**: Staging configs

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

## Next Steps

1. Implement actual validation logic in each action
2. Integrate with your release management system
3. Add actual deployment commands
4. Configure GitHub environments with protection rules
5. Add secrets and environment variables as needed
6. Set up notifications for deployment status
7. Add rollback workflows if needed

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
