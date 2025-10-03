# Setting Up Approval Gates for Release Promotions

This guide explains how to configure GitHub environment protection rules to enable manual approvals for release promotions.

## Overview

The promotion workflow includes built-in approval gates that pause execution and require manual review before promoting releases to higher environments. This ensures:

- **Change Control**: Human oversight of production changes
- **Risk Mitigation**: Additional validation before critical deployments
- **Compliance**: Audit trail of who approved what and when
- **Flexibility**: Different approval requirements per environment

## Quick Setup

### Step 1: Access Environment Settings

1. Go to your GitHub repository
2. Click **Settings** → **Environments**
3. If you don't see "Environments", enable GitHub Actions in **Settings** → **Actions** → **General**

### Step 2: Create Promotion Approval Environments

Create the following environments for approval gates:

#### Environment: `promotion-approval-staging`

This environment controls approvals for promoting to staging.

**Configuration:**
```
Environment name: promotion-approval-staging

Protection rules:
✓ Required reviewers: 1
  Reviewers: @dev-team-leads, @platform-team

□ Wait timer: 0 minutes
□ Deployment branches: All branches
□ Allow administrators to bypass protection rules
```

**Recommended reviewers:**
- Development team leads
- Platform engineers
- QA leads

#### Environment: `promotion-approval-production`

This environment controls approvals for promoting to production.

**Configuration:**
```
Environment name: promotion-approval-production

Protection rules:
✓ Required reviewers: 2
  Reviewers: @senior-engineers, @engineering-managers, @product-owners

□ Wait timer: 0 minutes (or set delay if required)
□ Deployment branches: All branches
□ Allow administrators to bypass protection rules (not recommended)
```

**Recommended reviewers:**
- Senior engineers
- Engineering managers
- Product owners
- DevOps/SRE team leads

**Note**: Requiring 2 reviewers for production adds an extra layer of safety.

### Step 3: Create Deployment Target Environments

Also create environments for actual deployments (optional but recommended):

#### Environment: `dev`
```
Environment name: dev

Protection rules: (minimal or none)
□ No required reviewers (dev is typically auto-deployed)

Secrets:
- DEV_AWS_ACCESS_KEY
- DEV_DATABASE_URL
- etc.

Variables:
- API_URL=https://api-dev.example.com
- LOG_LEVEL=debug
```

#### Environment: `staging`
```
Environment name: staging

Protection rules:
✓ Required reviewers: 1 (optional, for direct staging deployments)
  Reviewers: @platform-team

Secrets:
- STAGING_AWS_ACCESS_KEY
- STAGING_DATABASE_URL
- etc.

Variables:
- API_URL=https://api-staging.example.com
- LOG_LEVEL=info
```

#### Environment: `production`
```
Environment name: production

Protection rules:
✓ Required reviewers: 2
  Reviewers: @senior-engineers, @sre-team

✓ Deployment branches: Selected branches
  Allowed branches: main, release/*

Secrets:
- PROD_AWS_ACCESS_KEY
- PROD_DATABASE_URL
- etc.

Variables:
- API_URL=https://api.example.com
- LOG_LEVEL=warn
```

## Detailed Configuration Guide

### Adding Reviewers

1. In environment settings, click **Add protection rule**
2. Check **Required reviewers**
3. Click **Add reviewers**
4. Search for:
   - Individual users: `@username`
   - Teams: `@org/team-name`
5. Select up to 6 reviewers/teams
6. Set number of required reviewers (1-6)

**Best Practices:**
- Use teams rather than individuals when possible
- Have multiple eligible reviewers to avoid bottlenecks
- Higher environments = more reviewers required
- Include people across different time zones for 24/7 coverage

### Setting Wait Timers

Wait timers add a mandatory delay before deployment can proceed:

```
□ Wait timer: 30 minutes
```

**Use cases:**
- **Production deployments**: 15-30 minute delay to allow last-minute cancellation
- **Off-hours deployments**: Prevent accidental overnight deployments
- **Change freeze periods**: Enforce business rules

**Note**: Wait timer starts AFTER approval is granted.

### Deployment Branch Restrictions

Control which branches can deploy to each environment:

#### All branches (default)
```
□ Deployment branches: All branches
```
- Any branch can trigger deployment
- Good for dev/staging environments

#### Selected branches
```
✓ Deployment branches: Selected branches
  Branch patterns:
  - main
  - master
  - release/*
  - hotfix/*
```
- Only specified branches can deploy
- **Recommended for production**
- Use wildcard patterns for flexibility

### Environment Secrets

Store sensitive credentials per environment:

1. In environment settings, click **Add Secret**
2. Add secrets like:
   - `AWS_ACCESS_KEY_ID`
   - `AWS_SECRET_ACCESS_KEY`
   - `DATABASE_URL`
   - `API_KEY`
   - `SIGNING_KEY`

**Advantages over repository secrets:**
- Different credentials per environment
- More granular access control
- Easier rotation and management

### Environment Variables

Store non-sensitive configuration:

1. In environment settings, click **Add Variable**
2. Add variables like:
   - `API_URL`
   - `LOG_LEVEL`
   - `FEATURE_FLAGS`
   - `MAX_CONNECTIONS`

**Benefits:**
- Environment-specific configuration
- Visible in UI (unlike secrets)
- Easy to update without code changes

## Approval Workflow in Action

### How It Works

When someone triggers a promotion to production:

1. **Workflow starts**: `promote-release.yml` begins execution
2. **Validation runs**: Automatic checks complete
3. **Pause for approval**: Workflow reaches approval gate
4. **Notification sent**: GitHub notifies configured reviewers
5. **Review UI available**: Reviewers see promotion details
6. **Decision made**: Reviewer approves or rejects
7. **Execution continues**: If approved, promotion proceeds

### Reviewer Experience

Reviewers will see:

```
Review pending deployments
━━━━━━━━━━━━━━━━━━━━━━━━━

Workflow: Promote Release Between Environments
Environment: promotion-approval-production
Requested by: @developer-name
Waiting since: 2 minutes ago

Deployment details:
- Release ID: release-20250103-120000
- From: staging
- To: production

[Approve and deploy] [Reject]
```

**Reviewers can:**
- View full workflow run details
- See all job outputs and logs
- Check promotion request context
- Leave comments before deciding
- Approve or reject the promotion

### Approval Notifications

Reviewers are notified via:
- **GitHub notifications** (web + mobile app)
- **Email** (if enabled in GitHub settings)
- **Slack/Teams** (via GitHub integrations)

To ensure reviewers are notified:
1. Reviewers must have repository access
2. Reviewers should enable GitHub notifications
3. Consider adding Slack/Teams integration for critical environments

## Advanced Configuration

### Multiple Approval Stages

For extra-critical deployments, add multiple approval environments:

```yaml
jobs:
  security-approval:
    environment: security-approval
    # Security team approval
  
  business-approval:
    environment: business-approval
    needs: security-approval
    # Business stakeholder approval
  
  final-deployment:
    environment: production
    needs: business-approval
    # Actual deployment
```

### Conditional Approvals

Require approval only for certain conditions:

```yaml
jobs:
  approval:
    environment: ${{ inputs.target_environment == 'production' && 'approval-required' || 'no-approval' }}
```

### Time-Based Restrictions

Prevent deployments during certain hours:

```yaml
- name: Check Deployment Window
  run: |
    hour=$(date +%H)
    if [ $hour -ge 23 ] || [ $hour -le 6 ]; then
      echo "Deployments not allowed between 11 PM and 6 AM"
      exit 1
    fi
```

### Auto-Approval for Lower Environments

Skip approval for dev environment:

```yaml
approval:
  if: ${{ inputs.target_environment != 'dev' }}
  environment: promotion-approval-${{ inputs.target_environment }}
```

## Testing Your Setup

### Test Approval Flow

1. Trigger a promotion to staging:
   ```
   release_id: test-release-001
   source_environment: dev
   target_environment: staging
   ```

2. Verify workflow pauses at approval gate
3. Check that reviewers receive notification
4. Have reviewer approve/reject
5. Confirm workflow continues/stops appropriately

### Verify Environment Protection

1. Try to bypass approval (should fail)
2. Test with non-authorized user (should be blocked)
3. Test deployment from unauthorized branch (should fail)
4. Verify secrets are available in deployment jobs

## Troubleshooting

### "Deployment is waiting for approval"

**Normal behavior!** This means the approval gate is working.

**Action**: Wait for authorized reviewer to approve.

### "Environment protection rules not met"

**Cause**: User attempting to deploy doesn't have permission.

**Solution**: 
- Add user to environment reviewers
- Or have authorized reviewer approve

### "This environment requires approval from 2 reviewers"

**Cause**: Only 1 reviewer approved (production requires 2).

**Solution**: Wait for second reviewer to approve.

### Approvers not notified

**Cause**: Notification settings or repository access issues.

**Solutions**:
1. Verify reviewers have repository access
2. Check GitHub notification settings
3. Enable email notifications
4. Add Slack/Teams integration
5. Pin important repositories

### Approval button not visible

**Cause**: User is not an authorized reviewer.

**Solution**: Only users/teams configured as environment reviewers can approve.

## Security Best Practices

1. **Least Privilege**: Only add necessary reviewers
2. **Separation of Duties**: Different reviewers for different environments
3. **Two-Person Rule**: Require 2 approvals for production
4. **Branch Protection**: Restrict production deployments to main/release branches
5. **Audit Logging**: Review approval history regularly
6. **Rotate Reviewers**: Update reviewer list as team changes
7. **No Admin Bypass**: Don't allow admins to bypass protection rules
8. **Environment Secrets**: Use environment-specific secrets, not global ones

## Compliance and Audit

### Viewing Approval History

1. Go to **Settings** → **Environments** → [environment name]
2. Click **Deployment history**
3. View all deployments with:
   - Who requested
   - Who approved
   - When approved
   - Outcome

### Generating Audit Reports

Use GitHub API to export approval data:

```bash
curl -H "Authorization: token $GITHUB_TOKEN" \
  https://api.github.com/repos/$OWNER/$REPO/deployments
```

### Compliance Requirements

This setup helps meet compliance requirements for:
- **SOC 2**: Change management controls
- **ISO 27001**: Access control and segregation of duties
- **PCI DSS**: Change control procedures
- **HIPAA**: Audit trails and access controls
- **GDPR**: Data processing controls

## Examples

### Example 1: Standard Three-Tier Setup

```
Environments:
├── dev (no approval)
├── promotion-approval-staging (1 reviewer: dev leads)
├── staging (optional approval for direct deploys)
├── promotion-approval-production (2 reviewers: senior engineers + managers)
└── production (2 reviewers for any direct changes)

Flow:
dev → [auto] → staging → [1 approval] → production → [2 approvals]
```

### Example 2: High-Security Setup

```
Environments:
├── dev (no approval)
├── promotion-approval-staging (1 reviewer)
├── staging (1 reviewer)
├── security-approval (1 security team reviewer)
├── promotion-approval-production (2 reviewers)
└── production (2 reviewers + wait timer)

Flow:
dev → staging → [1 approval] → security-review → [1 approval] → production → [2 approvals + 30 min wait]
```

### Example 3: Continuous Deployment

```
Environments:
├── dev (auto-deploy from feature branches)
├── staging (auto-deploy from main)
├── promotion-approval-production (2 reviewers, business hours only)
└── production (2 reviewers, main branch only)

Flow:
feature/* → [auto] → dev
main → [auto] → staging
staging → [2 approvals] → production
```

## Additional Resources

- [GitHub Environments Documentation](https://docs.github.com/en/actions/deployment/targeting-different-environments)
- [Environment Protection Rules](https://docs.github.com/en/actions/deployment/targeting-different-environments/using-environments-for-deployment)
- [Required Reviewers](https://docs.github.com/en/actions/deployment/targeting-different-environments/using-environments-for-deployment#required-reviewers)

## Support

If you need help setting up approvals:

1. Check GitHub Actions logs for specific error messages
2. Verify environment configuration in repository settings
3. Test with a non-production environment first
4. Review this guide's troubleshooting section
5. Consult your DevOps/Platform team

