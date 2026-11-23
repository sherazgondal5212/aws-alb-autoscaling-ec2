# Gondal Digital Agency
## Cloud & DevOps Engineering | Portfolio Project 2.2

**Project status:** Tested · AWS cleanup complete · GitHub publication pending

# EC2 Behind an Application Load Balancer with Auto Scaling

A self-healing, load-balanced web tier on AWS that scaled from 2 to 4 EC2 instances under sustained load, with the results supported by terminal output and console evidence.

## Project at a glance

| Metric | Result |
|---|---:|
| Initial capacity | 2 EC2 instances |
| Scaled capacity | 4 EC2 instances |
| Instance type | `t3.micro` |
| AWS Region | `us-east-1` |
| Availability Zones observed | `us-east-1b`, `us-east-1c`, `us-east-1d`, `us-east-1f` |
| Healthy targets after scale-out | 4 of 4 |
| Replacement launch initiated | Within 9 seconds of termination starting |
| Approximate time from sustained load to scale-out | 5 minutes |
| Live demo | Not available; AWS stack was torn down to avoid ongoing charges |

## Architecture

Internet traffic enters through one Application Load Balancer (ALB). EC2 instances accept inbound web traffic only from the ALB security group.

```text
                         Internet
                            |
                         HTTP :80
                            |
                 Application Load Balancer
                    SG: gda-p22-alb-sg
                            |
                       Target Group
                            |
        Auto Scaling Group (minimum 2, maximum 4)
                    SG: gda-p22-ec2-sg
                            |
       +--------------------+--------------------+
       |                    |                    |
    EC2 t3.micro         EC2 t3.micro         EC2 t3.micro         EC2 t3.micro
    us-east-1b           us-east-1c           us-east-1d           us-east-1f
       |                    |                    |                    |
       +--------------------+--------------------+--------------------+
                            |
           Amazon Linux 2023 · Apache · IMDSv2
```

**Scaling configuration**

- Policy type: target tracking
- Metric: `ALBRequestCountPerTarget`
- Target value: `30`
- CloudWatch alarms: `AlarmHigh` and `AlarmLow` (created by the target-tracking policy)

## Terminal evidence

Evidence below was copied from the AWS CloudShell session on **23 November 2026 (UTC)**.

### Test A — Load balancing

Two healthy targets served requests through the ALB from two Availability Zones.

| Instance ID | Target health | Availability Zone | Requests observed |
|---|---|---|---:|
| `i-080e0cf83dbca007c` | Healthy | `us-east-1d` | 6 of 10 |
| `i-0bc737e629efbcee2` | Healthy | `us-east-1b` | 4 of 10 |

The observed split was 6/4. This demonstrates traffic distribution across both instances; it is **not** described as perfect round-robin balancing.

### Test B — Self-healing

The Auto Scaling Group detected a terminating instance and initiated a replacement.

```text
12:14:23  Terminating EC2 instance: i-080e0cf83dbca007c
          WaitingForELBConnectionDraining

12:14:32  Launching a new EC2 instance: i-04c2d3524d12c070a
```

A subsequent target-health check showed:

| Instance ID | Target state |
|---|---|
| `i-080e0cf83dbca007c` | Draining |
| `i-04c2d3524d12c070a` | Healthy |
| `i-0bc737e629efbcee2` | Healthy |

**Result:** The ASG started launching a replacement 9 seconds after the termination event began. The replacement later became healthy without manual intervention. The other instance remained healthy.

### Test C — Scale-out

A sustained load test ran for approximately 7 minutes. The `AlarmHigh` alarm crossed its threshold:

```text
ALARM
Threshold Crossed: 3 datapoints
[442066.5 (12:32), 439754.0 (12:31), 122052.0 (12:30)]
were greater than the threshold (30.0).
```

The Auto Scaling Group capacity changed from 2 to 4:

```text
Desired capacity: 4
Current capacity: 4

12:35:59  Launching a new EC2 instance: i-05de71f0608a15c57
12:35:59  Launching a new EC2 instance: i-09f829f674374ca5a
```

The target group subsequently showed **4 total targets, 4 healthy, 0 unhealthy**, distributed across `us-east-1b`, `us-east-1c`, `us-east-1d`, and `us-east-1f`.

The ASG activity history confirmed that `AlarmHigh` triggered `gda-p22-req-policy`, increasing capacity from 2 to 4.

## Technical workflow

The infrastructure was built using AWS CloudShell and the AWS CLI.

1. **Discover defaults:** Identified the default VPC, default subnets across Availability Zones, and the Amazon Linux 2023 AMI through SSM Parameter Store.
2. **Create least-privilege security groups:** The ALB security group allowed HTTP port 80 from the internet. The EC2 security group allowed HTTP port 80 only from the ALB security group.
3. **Configure launch template:** Used `t3.micro`, required IMDSv2, and supplied user data to install Apache and write the instance ID and Availability Zone to the web page.
4. **Create target group, ALB, and listener:** Configured an HTTP listener and an HTTP health check on `/`, with a 15-second interval and 2 successful checks for healthy status.
5. **Configure Auto Scaling Group:** Set minimum capacity to 2, maximum capacity to 4, ELB health checks, a 120-second health-check grace period, and subnets across the default Availability Zones.
6. **Configure target tracking:** Set `ALBRequestCountPerTarget` to 30. CloudWatch `AlarmHigh` and `AlarmLow` alarms were created by the scaling policy.
7. **Test and clean up:** Verified load distribution, self-healing, and scale-out, then removed the project infrastructure to avoid ongoing AWS charges.

## Testing summary

| Test | What was checked | Result |
|---|---|---|
| A. Load balancing | 10 requests through the ALB DNS name | **Passed.** Two instances in two AZs served traffic; observed split was 6 and 4. |
| B. Self-healing | Terminated one instance and observed ASG replacement | **Passed.** Replacement launch began within 9 seconds; it later became healthy without manual intervention. |
| C. Scale-out | Approximately 7 minutes of sustained load | **Passed.** `AlarmHigh` triggered, capacity increased from 2 to 4, and all 4 targets became healthy. |

### Lesson learned: sustained load matters

The first load test (`ab -n 200000`) completed in about one minute. That was insufficient for the alarm to observe the three consecutive one-minute datapoints needed to trigger the scaling policy. The metric showed approximately 100,000 requests per target in one minute (200,000 total), followed by silence.

Using a time-based load test (`-t 420`) sustained traffic long enough for the alarm and scaling policy to respond.

## Automation scripts

The following scripts are **planned for the repository** and are to be derived from the commands used during the original deployment. They are not represented as existing or validated automation scripts.

### `scripts/deploy.sh` — planned

Intended to automate variables, security groups, launch template, target group, ALB, listener, Auto Scaling Group, and scaling policy creation.

### `scripts/cleanup.sh` — planned

Intended to remove the Auto Scaling Group, listener, ALB, target group, launch template, and security groups, including waits for dependent resources.

### Operational notes

- Set `AWS_PAGER=""` to prevent AWS CLI pager interruptions.
- Allow approximately 2 minutes after ALB deletion before attempting target-group deletion, so dependent resources can finish releasing.

## Project outcomes and metrics

| Outcome | Observed result |
|---|---:|
| Scale-out | 2 → 4 instances |
| Healthy targets after scale-out | 4 of 4 |
| Availability Zones | 4 |
| Time to initiate replacement launch | 9 seconds from termination start |
| Approximate time from load start to scale-out | 5 minutes |
| Observed requests per target per minute | Approximately 392k–442k (12:31–12:36 UTC) |
| Deregistration delay | 300 seconds (default; unchanged) |

## Limitations and next steps

### Limitations

- Used the default VPC with public EC2 instances.
- HTTP only; HTTPS was not configured.
- No request-level downtime test was performed during instance replacement.
- The Apache Benchmark (`ab`) summary metrics, including failed requests and requests per second, were not captured.
- No live demo is available because the AWS stack has been removed to avoid ongoing charges.

### Next project: Project 2.2b

The planned next iteration will use Terraform, a custom VPC, private subnets, and HTTPS through AWS Certificate Manager (ACM).

## GitHub and live links

| Resource | Link / status |
|---|---|
| GitHub profile | [github.com/sherazgondal5212](https://github.com/sherazgondal5212) |
| Repository | `aws-alb-autoscaling-ec2` — publication pending |
| Live demo | None — AWS resources have been torn down |

## About Gondal Digital Agency

**Gondal Digital Agency (GDA)** — premium international cloud and digital engineering, supporting startups and businesses with reliable technology solutions on AWS.

Website: [gondaldigitalagency.com](https://gondaldigitalagency.com)

**Muhammad Sheraz Gondal**  
Founder & CEO, Gondal Digital Agency
