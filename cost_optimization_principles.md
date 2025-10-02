# Cost Optimization Principles

**Universal patterns for building cost-effective systems**

These principles apply to any infrastructure with ongoing costs: cloud platforms, SaaS services, compute resources, or managed services.

---

## 1. Cost-First Infrastructure Design

### Principle

**Always perform cost analysis during the planning phase, before implementation.**

Cost isn't an afterthought—it's a first-class design constraint, just like performance or security.

### Why This Matters

- **Prevents expensive surprises** - Know costs before you commit
- **Enables informed decisions** - Choose architectures with eyes open
- **Catches expensive patterns early** - Fix before deployment, not after
- **Creates cost awareness** - Team understands resource implications

### The Rule

```
BEFORE implementing infrastructure:
    1. Estimate monthly costs for each component
    2. Compare cost vs value for each service
    3. Document cost implications of decisions
    4. Get approval for costs above threshold
THEN implement
```

### Examples

#### ❌ Bad Practice
```yaml
# Deploy first, discover costs later
Resources:
  Database:
    Type: AWS::RDS::DBInstance
    Properties:
      DBInstanceClass: db.r5.24xlarge  # 768GB RAM, $13,000/month
      MultiAZ: true
      # Deployed without cost review
```

**Problems:**
- $13,000/month database deployed without review
- Could have started smaller and scaled up
- No business justification documented
- Cost discovered in first bill

#### ✅ Good Practice
```yaml
# Cost analysis before deployment
# COST ANALYSIS (as of 2025-10):
# - Dev Environment: db.t3.micro @ $15/month
# - Production: db.t3.small @ $30/month (start small)
# - Scale trigger: When 80% CPU sustained > 1 hour
# - Max justified cost: $500/month without additional approval
# - Annual projected: $360-$600 depending on growth

Resources:
  Database:
    Type: AWS::RDS::DBInstance
    Properties:
      DBInstanceClass: !If [IsProduction, db.t3.small, db.t3.micro]
      MultiAZ: !If [IsProduction, true, false]
      # Start small, scale based on metrics
```

**Benefits:**
- Costs known before deployment
- Environment-appropriate sizing
- Scale triggers defined
- Approval process for larger instances

### Cost Analysis Template

```markdown
## Infrastructure Cost Analysis

**Component:** [Name of service/resource]
**Estimated Monthly Cost:**
- Development: $X
- Staging: $Y
- Production: $Z

**Cost Breakdown:**
- Compute: $A
- Storage: $B
- Data transfer: $C
- Other: $D

**Justification:**
- Why this tier/size?
- What cheaper alternatives were considered?
- What's the cost/value trade-off?

**Scale Triggers:**
- When do we need to size up?
- What metrics indicate need for more resources?

**Annual Projection:** $MIN - $MAX
**Approval:** [Name] approved [Date]
```

---

## 2. Pay-Per-Use Default

### Principle

**Default to pay-per-use resources. Only add fixed costs when measured need exists.**

Fixed costs are easy to add but hard to remove. Start with variable costs that scale with actual usage.

### Why This Matters

- **Lower risk** - No monthly bills when not used
- **Scales with value** - Costs grow with usage/revenue
- **Easier to justify** - Paying for what you use, not what you might need
- **Better for experiments** - Try ideas without committing to ongoing costs

### The Rule

```
IF resource_supports_pay_per_use THEN
    start_with_pay_per_use
    monitor_actual_usage
    IF usage_consistent_and_high THEN
        consider_reserved_capacity
    END IF
ELSE
    start_with_smallest_fixed_tier
END IF
```

### Examples

#### Compute

**❌ Bad: Start with always-on servers**
```yaml
# EC2 instance running 24/7 for occasional batch jobs
EC2Instance:
  Type: AWS::EC2::Instance
  Properties:
    InstanceType: t3.large  # $60/month running 24/7
```
**Cost:** $60/month even if used 1 hour/day

**✅ Good: Start with Lambda (pay-per-invocation)**
```yaml
# Lambda for batch jobs - pay only when running
BatchFunction:
  Type: AWS::Lambda::Function
  Properties:
    Runtime: python3.11
    MemorySize: 3008  # 3GB
    Timeout: 900      # 15 min max
```
**Cost:** $0.20/hour when running → $6/month for 1 hour/day

#### Database

**❌ Bad: Start with provisioned database**
```yaml
# Provisioned RDS running 24/7
Database:
  Type: AWS::RDS::DBInstance
  Properties:
    DBInstanceClass: db.t3.small  # $30/month always on
```
**Cost:** $30/month even for dev environments with sporadic use

**✅ Good: Start with serverless database**
```yaml
# Aurora Serverless v2 - scales to zero
Database:
  Type: AWS::RDS::DBCluster
  Properties:
    Engine: aurora-postgresql
    ServerlessV2ScalingConfiguration:
      MinCapacity: 0.5  # Scales down when idle
      MaxCapacity: 1    # Scales up when needed
```
**Cost:** ~$40/month for production, ~$5/month for dev (only when active)

### When to Switch to Fixed Costs

**Indicators you're ready for reserved capacity:**

1. **Consistent high usage** - Running 24/7 for 30+ days
2. **Cost threshold reached** - Pay-per-use costs exceed reserved costs
3. **Predictable demand** - Usage patterns are stable
4. **Business commitment** - Feature proven valuable, not an experiment

**Calculate the breakeven:**
```
Monthly pay-per-use cost: $X
Reserved instance cost: $Y

If X > Y for 3+ consecutive months → Consider reserved
```

---

## 3. Environment-Appropriate Sizing

### Principle

**Use environment-appropriate resource sizing. Don't use production resources for development.**

Development and staging environments should use minimal resources that support development work—not production-scale infrastructure.

### Why This Matters

- **Massive cost savings** - Dev/staging can be 90% cheaper
- **Faster iteration** - Smaller resources provision faster
- **Realistic testing** - Stage with production-sized data, not infrastructure
- **Focus spending** - Put budget toward user-facing systems

### The Rule

```
Development:   Minimal resources, fast iteration, low cost
Staging:       Production-like data, smaller infrastructure
Production:    Sized for actual load, monitored, scaled as needed
```

### Examples

#### ❌ Bad: Same Size Everywhere
```yaml
# Same database for all environments
Database:
  Type: AWS::RDS::DBInstance
  Properties:
    DBInstanceClass: db.r5.xlarge  # $365/month × 3 environments = $1,095/month
    MultiAZ: true
```

**Costs:**
- Dev: $365/month (wasted, used 10% of capacity)
- Staging: $365/month (wasted, used 20% of capacity)
- Production: $365/month (appropriate)
- **Total: $1,095/month**

#### ✅ Good: Environment-Appropriate Sizing
```yaml
Database:
  Type: AWS::RDS::DBInstance
  Properties:
    DBInstanceClass:
      !If [IsProd, db.r5.xlarge, !If [IsStaging, db.t3.small, db.t3.micro]]
    MultiAZ:
      !If [IsProd, true, false]
```

**Costs:**
- Dev: $15/month (db.t3.micro, single-AZ)
- Staging: $30/month (db.t3.small, single-AZ)
- Production: $365/month (db.r5.xlarge, multi-AZ)
- **Total: $410/month (62% savings)**

### Sizing Guidelines

| Environment | Purpose | Sizing Strategy | Availability |
|------------|---------|-----------------|--------------|
| **Development** | Fast iteration, bug fixing | Smallest tier that works | Single-AZ, no HA |
| **Staging** | Pre-prod testing, QA | Small tier, production data subset | Single-AZ |
| **Production** | Real users | Sized for actual load + headroom | Multi-AZ, HA |

**Development Environment Goals:**
- Fast deployment (< 5 min)
- Low cost (< $50/month total)
- Good enough performance for debugging
- Representative of prod architecture

**NOT Development Environment Goals:**
- ❌ Production-scale performance
- ❌ High availability
- ❌ Large datasets
- ❌ Multi-region

---

## 4. Cost Analysis Gates

### Principle

**Validate infrastructure costs before deployment. Require approval for costs above threshold.**

Automated cost validation prevents expensive mistakes from reaching production.

### Why This Matters

- **Catches mistakes early** - Wrong instance type caught before deployment
- **Enforces budgets** - Can't exceed budget without approval
- **Creates accountability** - Costs are reviewed, not blind
- **Documents decisions** - Cost approvals create audit trail

### The Rule

```
BEFORE deployment:
    estimate_monthly_cost()
    IF cost > threshold THEN
        require_approval()
        document_justification()
    END IF
    validate_environment_appropriate_sizing()
THEN deploy
```

### Implementation Examples

#### Pre-Deployment Cost Check (CI/CD)
```yaml
# GitHub Actions workflow
name: Cost Analysis Gate
on: [pull_request]

jobs:
  cost-check:
    runs-on: ubuntu-latest
    steps:
      - name: Estimate Infrastructure Costs
        run: |
          # Use infracost or similar tool
          infracost breakdown --path . --format json > costs.json

      - name: Check Cost Threshold
        run: |
          monthly_cost=$(jq '.totalMonthlyCost' costs.json)

          if (( $(echo "$monthly_cost > 100" | bc -l) )); then
            echo "❌ Monthly cost $monthly_cost exceeds $100 threshold"
            echo "Requires approval from team lead"
            exit 1
          fi

          echo "✅ Monthly cost $monthly_cost within threshold"
```

#### CloudFormation Stack Policy
```yaml
# Prevent expensive resources without review
StackPolicy:
  Statement:
    - Effect: Deny
      Principal: "*"
      Action: "Update:*"
      Resource: "*"
      Condition:
        StringEquals:
          ResourceType:
            - AWS::RDS::DBInstance
            - AWS::EC2::Instance
          # Require manual review for databases and instances
```

#### Cost Approval Process
```markdown
## Infrastructure Change Request

**Component:** New PostgreSQL database
**Estimated Monthly Cost:** $45/month
**Environment:** Production
**Justification:** User data storage for new feature

**Cost Breakdown:**
- db.t3.small: $30/month
- 100GB storage: $10/month
- Backups: $5/month

**Alternatives Considered:**
- Shared database: Would create coupling between services
- Larger instance: Overkill for initial launch
- Serverless: Not available for PostgreSQL in our region

**Approval:** [@tech-lead] LGTM, within budget
**Deployment Date:** 2025-10-15
```

### Cost Thresholds

Set thresholds based on your organization:

**Example Thresholds:**
- < $50/month: Auto-approve
- $50-200/month: Team lead approval
- $200-1000/month: Engineering manager approval
- > $1000/month: Director approval + business justification

---

## 5. Right-Size for Actual Load

### Principle

**Size resources based on measured load, not guesses. Monitor and adjust continuously.**

### Why This Matters

- **Avoid over-provisioning** - Don't pay for unused capacity
- **Avoid under-provisioning** - Don't impact user experience
- **Adapt to change** - Usage patterns change over time
- **Data-driven decisions** - Metrics, not hunches

### The Rule

```
WHILE system_running:
    measure_actual_usage()
    IF consistently_below_threshold THEN
        consider_downsizing()
    ELSE IF consistently_above_threshold THEN
        consider_upsizing()
    END IF
    document_change_rationale()
END WHILE
```

### Example: Database Sizing

#### Initial Deployment (Conservative)
```yaml
# Start with small, known-good configuration
Database:
  Type: AWS::RDS::DBInstance
  Properties:
    DBInstanceClass: db.t3.small  # $30/month
    AllocatedStorage: 20          # 20GB
```

#### After 30 Days of Monitoring
```
Metrics observed:
- Average CPU: 15%
- Peak CPU: 45%
- Average connections: 8
- Storage used: 3GB
- Query performance: All queries < 100ms

Analysis: Over-provisioned
- CPU well below 80% threshold
- Storage has 6x headroom
- Performance excellent

Action: Downsize to db.t3.micro ($15/month)
Savings: $15/month (50% reduction)
```

#### After 6 Months (Growth)
```
Metrics observed:
- Average CPU: 65%
- Peak CPU: 92%
- Average connections: 45
- Storage used: 15GB
- Query performance: Some queries > 500ms during peaks

Analysis: Approaching capacity limits
- CPU peaks concerning (> 90%)
- Storage still OK (20% free)
- Performance degrading during peaks

Action: Upsize to db.t3.medium ($60/month)
Justification: User growth = 3x, need better performance
```

### Monitoring Checklist

For every billable resource, monitor:
- [ ] Utilization metrics (CPU, memory, storage)
- [ ] Performance metrics (latency, throughput)
- [ ] Cost trends (monthly costs, per-unit costs)
- [ ] Usage patterns (peak vs average, time of day)

**Review frequency:**
- New resources: Weekly for first month
- Established resources: Monthly
- Large costs (>$500/mo): Weekly

---

## 6. Eliminate Waste

### Principle

**Actively look for and eliminate wasteful spending.**

### Common Sources of Waste

#### 1. Orphaned Resources
```bash
# Find unattached EBS volumes
aws ec2 describe-volumes \
  --filters Name=status,Values=available \
  --query 'Volumes[*].[VolumeId,Size,CreateTime]'

# Cost: $0.10/GB/month per unattached volume
# Example: 50GB forgotten volume = $5/month wasted
```

#### 2. Over-Provisioned Resources
```bash
# Find EC2 instances with <10% CPU utilization
aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value=i-1234567890abcdef0 \
  --statistics Average \
  --start-time 2025-10-01T00:00:00Z \
  --end-time 2025-10-31T23:59:59Z \
  --period 86400

# If average < 10%: Downsize or consider serverless
```

#### 3. Forgotten Test Resources
```yaml
# Add auto-expiration tags to test resources
Tags:
  - Key: Environment
    Value: test
  - Key: AutoDelete
    Value: "2025-10-30"  # Delete after 30 days
  - Key: Owner
    Value: john@example.com
```

#### 4. Excessive Data Transfer
```bash
# Monitor data transfer costs
aws ce get-cost-and-usage \
  --time-period Start=2025-10-01,End=2025-10-31 \
  --granularity MONTHLY \
  --metrics BlendedCost \
  --filter file://filter.json

# High data transfer costs often indicate:
# - Unnecessary cross-region calls
# - Inefficient data fetching
# - Missing CDN/caching
```

### Monthly Cost Review Checklist

- [ ] Review all resources > $10/month
- [ ] Check for orphaned/unused resources
- [ ] Verify resource utilization (look for < 20% usage)
- [ ] Review data transfer costs for optimization opportunities
- [ ] Check for forgotten test/dev resources
- [ ] Validate environment sizing is still appropriate
- [ ] Update cost forecasts based on growth

---

## Summary

Apply these six principles to keep infrastructure costs under control:

1. **Cost-First Infrastructure Design** → Know costs before you commit
2. **Pay-Per-Use Default** → Start variable, go fixed when proven
3. **Environment-Appropriate Sizing** → Don't use production resources for dev
4. **Cost Analysis Gates** → Prevent expensive mistakes with validation
5. **Right-Size for Actual Load** → Monitor and adjust based on metrics
6. **Eliminate Waste** → Actively find and remove unnecessary costs

**Start with these principles from day one.** Retrofitting cost optimization is much harder than building it in from the start.

---

**Questions?** Contact info@happyhippo.ai
