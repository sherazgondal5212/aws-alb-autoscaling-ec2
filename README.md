# EC2 Behind an Application Load Balancer with Auto Scaling

**Cloud & DevOps Engineering \| Portfolio Project 2.2**

![Project status](https://img.shields.io/badge/Status-Tested-success)
![AWS](https://img.shields.io/badge/Cloud-AWS-orange)
![Region](https://img.shields.io/badge/Region-us--east--1-blue)

A self-healing, load-balanced web tier on AWS. The project demonstrated
traffic distribution across EC2 instances, automatic replacement of a
terminated instance, and scale-out from **2 to 4 EC2 instances** under
sustained load.

> **Deployment status:** Tested successfully. The AWS project
> infrastructure was removed after testing to avoid ongoing charges.
> This repository contains the project documentation and evidence
> screenshots; it is not a live deployment.

## Project overview

  -----------------------------------------------------------------------
  Item                                Result
  ----------------------------------- -----------------------------------
  AWS Region                          `us-east-1`

  Operating system                    Amazon Linux 2023

  Instance type                       `t3.micro`

  Initial Auto Scaling capacity       2 instances

  Maximum Auto Scaling capacity       4 instances

  Healthy targets after scale-out     4 of 4

  Availability Zones observed         `us-east-1b`, `us-east-1c`,
                                      `us-east-1d`, `us-east-1f`

  Load balancer                       Application Load Balancer (ALB)

  Listener                            HTTP, port 80

  Web server                          Apache

  Instance metadata                   IMDSv2 required

  Live demo                           Not available; AWS resources have
                                      been removed
  -----------------------------------------------------------------------

## Architecture

Internet traffic enters through the Application Load Balancer. The ALB
forwards requests to its registered targets. The Auto Scaling Group
manages the EC2 fleet and registers replacement/new instances with the
target group.

``` text
                         Internet
                            |
                         HTTP :80
                            |
                 Application Load Balancer
                    SG: gda-p22-alb-sg
                            |
                       Target Group
                            |
             +--------------+--------------+
             |              |              |
          EC2 fleet managed by Auto Scaling Group
             |              |              |
          EC2 #1          EC2 #2        EC2 #3 / #4
          t3.micro        t3.micro       t3.micro
             |              |              |
             +--------------+--------------+
                            |
              Amazon Linux 2023 + Apache
                   IMDSv2 required
```

### Security configuration

-   The ALB security group (`gda-p22-alb-sg`) allowed inbound HTTP
    traffic on port 80.
-   The EC2 security group (`gda-p22-ec2-sg`) allowed inbound HTTP
    traffic on port 80 **only from the ALB security group**.
-   The launch template required IMDSv2.
-   The instances used a user-data script to install Apache and display
    instance metadata (instance ID and Availability Zone) on the web
    page.

### Auto Scaling configuration

  -----------------------------------------------------------------------
  Setting                             Configuration
  ----------------------------------- -----------------------------------
  Minimum capacity                    2

  Maximum capacity                    4

  Health check type                   ELB

  Health-check grace period           120 seconds

  Scaling policy                      Target tracking

  Scaling metric                      `ALBRequestCountPerTarget`

  Target value                        30 requests per target per minute

  CloudWatch alarms                   `AlarmHigh` and `AlarmLow` (created
                                      by the target-tracking policy)
  -----------------------------------------------------------------------

The target value is the configured target-tracking threshold, not a
claim that every individual minute remained at exactly 30 requests per
target.

## Test results and evidence

The following results were observed during the AWS CloudShell and EC2
console tests on **23 November 2026**. Screenshots are stored in the
`screenshots/` directory.

### Test 1 --- Load balancing

Two healthy targets in two Availability Zones served requests through
the ALB.

  Instance ID             Availability Zone     Requests observed
  ----------------------- ------------------- -------------------
  `i-080e0cf83dbca007c`   `us-east-1d`                    6 of 10
  `i-0bc737e629efbcee2`   `us-east-1b`                    4 of 10

The observed distribution was **6/4 across 10 requests**. This confirms
that both registered instances served traffic; it does not demonstrate a
strict or perfectly even round-robin distribution.

### Test 2 --- Auto Scaling self-healing

The Auto Scaling Group replaced an instance after its termination was
initiated.

  ------------------------------------------------------------------------
  Event                                   Time (UTC) Evidence
  --------------------- ---------------------------- ---------------------
  Original instance                         12:14:23 ASG activity history
  termination activity                               

  Replacement instance                      12:14:30 ASG activity history
  launch activity                                    

  Replacement instance         `i-04c2d3524d12c070a` Target later reported
                                                     Healthy
  ------------------------------------------------------------------------

**Observed result:** The ASG activity-history screenshot shows the
replacement launch activity at 12:14:30, seven seconds after the
termination activity at 12:14:23. A later target-health check showed the
replacement as Healthy. The seven-second figure refers to the **launch
activity being recorded**, not to the instance becoming healthy.

The original terminal notes recorded the launch at 12:14:32. The console
activity-history screenshot shows 12:14:30; this README uses the
screenshot timestamp for the documented seven-second comparison.

### Test 3 --- Scale-out under sustained load

A sustained load test ran for approximately seven minutes. The
target-tracking policy's `AlarmHigh` alarm entered the ALARM state and
triggered `gda-p22-req-policy`.

The alarm evidence included three datapoints above the configured
threshold of 30:

``` text
Threshold Crossed: 3 datapoints
[442066.5 (12:32), 439754.0 (12:31), 122052.0 (12:30)]
were greater than the threshold (30.0).
```

The Auto Scaling activity history recorded the capacity increase:

``` text
12:35:47  AlarmHigh triggered gda-p22-req-policy
12:35:57  Desired capacity increased from 2 to 4
```

Two new EC2 instances were launched:

-   `i-05de71f0608a15c57`
-   `i-09f829f674374ca5a`

The target-group screenshot subsequently showed **4 registered targets,
all Healthy**, across four Availability Zones:

-   `us-east-1b`
-   `us-east-1c`
-   `us-east-1d`
-   `us-east-1f`

The scale-out activity-history screenshot and target-health screenshot
are included in this repository's evidence folder.

### Test summary

  -----------------------------------------------------------------------
  Test                                Result
  ----------------------------------- -----------------------------------
  ALB traffic distribution            Passed --- both instances served
                                      requests; observed split 6/4

  Instance self-healing               Passed --- replacement launch
                                      activity recorded; replacement
                                      later became Healthy

  Target-tracking scale-out           Passed --- desired capacity
                                      increased from 2 to 4; all 4
                                      targets later Healthy
  -----------------------------------------------------------------------

## Technical workflow

The infrastructure was created using AWS CloudShell and the AWS CLI.

1.  **Discover AWS defaults:** Identified the default VPC, default
    subnets across Availability Zones, and the Amazon Linux 2023 AMI
    using SSM Parameter Store.
2.  **Create security groups:** Configured the ALB security group for
    inbound HTTP and the EC2 security group to accept HTTP only from the
    ALB security group.
3.  **Create launch template:** Configured `t3.micro`, required IMDSv2,
    and supplied user data to install Apache and display instance
    metadata.
4.  **Create load-balancing resources:** Created the target group, ALB,
    HTTP listener, and HTTP health check on `/`.
5.  **Create Auto Scaling Group:** Configured minimum capacity 2,
    maximum capacity 4, ELB health checks, a 120-second health-check
    grace period, and subnets across the selected default Availability
    Zones.
6.  **Configure target tracking:** Created a target-tracking policy
    using `ALBRequestCountPerTarget` with a target value of 30. AWS
    created the associated high/low CloudWatch alarms.
7.  **Test and clean up:** Tested request distribution, instance
    replacement, and scale-out. Removed the project infrastructure after
    testing to avoid ongoing AWS charges.

## Lesson learned: sustained load matters

The first load test (`ab -n 200000`) completed in approximately one
minute. That short burst did not sustain traffic long enough for the
target-tracking alarm to observe the required consecutive one-minute
datapoints.

A time-based load test (`-t 420`) maintained traffic for approximately
seven minutes and allowed the alarm and scaling policy to respond.

## Evidence screenshots

Add the supplied screenshots to the repository using these filenames:

  ------------------------------------------------------------------------------------
  Screenshot                                       Description
  ------------------------------------------------ -----------------------------------
  `screenshots/target-group-4-healthy.png`         Target group showing 4 registered
                                                   targets, all Healthy

  `screenshots/autoscaling-activity-history.png`   ASG activity history showing
                                                   instance replacement and scale-out
  ------------------------------------------------------------------------------------

Once the images are committed, the screenshots will render below:

### Target group --- 4 healthy targets

![Target group showing four healthy
targets](screenshots/target-group-4-healthy.png)

### Auto Scaling activity history

![Auto Scaling activity
history](screenshots/autoscaling-activity-history.png)

## Repository contents

``` text
aws-alb-autoscaling-ec2/
├── README.md
└── screenshots/
    ├── target-group-4-healthy.png
    └── autoscaling-activity-history.png
```

The two screenshots listed above are the evidence supplied for this
documentation update. Additional test screenshots may be added when
available.

## Limitations and future improvements

### Limitations of this implementation

-   Used the default VPC and public EC2 instances.
-   HTTP only; HTTPS was not configured.
-   No request-level downtime test was performed during instance
    replacement.
-   Apache Benchmark (`ab`) summary metrics, including failed requests
    and requests per second, were not captured.
-   The deployment was removed after testing, so there is no live demo.
-   Deployment and cleanup automation scripts are not included in this
    repository version.

### Planned next iteration --- Project 2.2b

The next iteration is planned to introduce:

-   Terraform infrastructure as code.
-   A custom VPC.
-   Private subnets for EC2 instances.
-   HTTPS using AWS Certificate Manager (ACM).
-   Improved deployment and cleanup automation.

## GitHub

-   **Repository:**
    [aws-alb-autoscaling-ec2](https://github.com/sherazgondal5212/aws-alb-autoscaling-ec2)
-   **GitHub profile:**
    [sherazgondal5212](https://github.com/sherazgondal5212)
-   **Live demo:** Not available --- AWS infrastructure has been
    removed.

## About Gondal Digital Agency

**Gondal Digital Agency (GDA)** is a cloud and digital engineering brand
focused on helping startups and businesses with technology solutions.

Website: [gondaldigitalagency.com](https://gondaldigitalagency.com/)

**Muhammad Sheraz Gondal**\
Founder & CEO, Gondal Digital Agency
