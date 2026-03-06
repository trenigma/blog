---
title: "How I Built a Production-Ready AWS Project to Reboot My DevOps Career"
date: 2025-10-27T10:00:00-08:00
draft: false
tags: ["aws", "terraform", "devops", "career"]
---

*After a career gap, I needed a portfolio project that would prove I could still build production-grade infrastructure. Here's the story of deploying a multi-AZ ECS application with Terraform - including the 7 real debugging issues I solved along the way.*

---

After taking time away from DevOps work, I knew I needed more than just an updated resume to get back into the field. I needed to prove I could still build real infrastructure, solve real problems, and think like a production engineer.

So I decided to go about building a project in my personal AWS account that I could use to help showcase DevOps work. I wanted to make something that I could use to emulate work I've done that someone else could actually use in production easily. After some mulling, I settled on an API using ECS managed with Terraform. 

**What I needed:**
- A project that demonstrated modern DevOps practices
- Infrastructure I could explain in technical interviews
- Real debugging I could discuss
- Something I could deploy and iterate on

**What I didn't want:**
- "Hello World" with extra steps
- Infrastructure that only works on my laptop
- Something I couldn't explain under pressure

## Architecture: Multi-AZ ECS with Terraform

I decided to build a REST API on AWS ECS Fargate, deployed across three availability zones, with complete Infrastructure as Code automation.

**The stack:**
- **Compute**: ECS Fargate with auto-scaling
- **Database**: RDS PostgreSQL with automated backups
- **Load Balancing**: Application Load Balancer with health checks
- **Networking**: VPC with public/private subnets across 3 AZs
- **Container Registry**: ECR with automated image scanning
- **Monitoring**: CloudWatch logs, metrics, and dashboards
- **IaC**: Terraform with modular design

**The application:**
A simple TODO REST API with full CRUD operations. Not because TODOs are exciting, but because they're familiar enough that interviewers can focus on the infrastructure, not the app logic.

Here's what the final architecture looks like:

<!-- ![architecture.png](https://blog.trenigma.dev/content/images/2025/11/architecture.png) -->

```
┌─────────────────────────────────────────────────────┐
│                    AWS Account                       │
│  ┌──────────────────────────────────────────────┐  │
│  │ VPC (10.0.0.0/16)                             │  │
│  │                                                │  │
│  │  ┌───────────┐  ┌───────────┐  ┌───────────┐ │  │
│  │  │Public AZ-1│  │Public AZ-2│  │Public AZ-3│ │  │
│  │  │ALB + NAT  │  │ALB + NAT  │  │    NAT    │ │  │
│  │  └───────────┘  └───────────┘  └───────────┘ │  │
│  │        │              │              │         │  │
│  │  ┌───────────┐  ┌───────────┐  ┌───────────┐ │  │
│  │  │Private AZ1│  │Private AZ2│  │Private AZ3│ │  │
│  │  │ECS Tasks  │  │ECS Tasks  │  │    RDS    │ │  │
│  │  └───────────┘  └───────────┘  └───────────┘ │  │
│  └──────────────────────────────────────────────┘  │
│                                                     │
│  Internet → ALB → ECS Fargate → RDS PostgreSQL    │
└─────────────────────────────────────────────────────┘
```


**Key decisions:**
- **ECS Fargate over EC2**: No server management, pay per task
- **Multi-AZ deployment**: Real high availability, not just buzzwords
- **Terraform modules**: Reusable components for networking, compute, database
- **Docker provider in Terraform**: Automated image builds (this was crucial)

## The Results: Portfolio Gold

**What I can now demonstrate:**
-  Multi-AZ architecture design
-  Infrastructure as Code with Terraform
-  Container orchestration with ECS
-  Debugging production issues systematically
-  Cost-conscious infrastructure decisions
-  Security best practices (private subnets, encrypted storage, secrets management)
-  Monitoring and observability setup

**What I can discuss in interviews:**
- Specific technical decisions and their trade-offs
- Real debugging methodologies
- Production vs development considerations
- Cost optimization strategies
- Security and compliance thinking

**What's deployed and working:**
- Live API: Fully functional with 200-300ms response times
- GitHub repo: Complete with comprehensive documentation
- CloudWatch: Real metrics and logs
- Infrastructure: Reproducible with a single command

## Fun Stuff I Ran Into During The Build


### Issue #1: ECR authentication fails

**The symptom:** ECS tasks failed to start with `ResourceInitializationError: unable to pull secrets or registry auth`.

**What I learned:** The ECS task execution role needs explicit ECR permissions, even though the documentation says `AmazonECSTaskExecutionRolePolicy` should cover it. Sometimes the managed policy isn't enough.

**The fix:** Added explicit ECR permissions to the task execution role:

```hcl
resource "aws_iam_role_policy" "ecs_task_execution_ecr" {
  name = "${var.project_name}-${var.environment}-ecs-ecr-policy"
  role = aws_iam_role.ecs_task_execution.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Action = [
        "ecr:GetAuthorizationToken",
        "ecr:BatchCheckLayerAvailability",
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage"
      ]
      Resource = "*"
    }]
  })
}
```


### Issue #2: RDS password fun

**The symptom:** Terraform apply failed with `The parameter MasterUserPassword is not a valid password. Only printable ASCII characters besides '/', '@', '"', ' ' may be used.`

**What I learned:** RDS has specific password constraints that aren't immediately obvious. The `random_password` resource in Terraform generates all sorts of special characters by default.

**The fix:** Constrained the character set:

```hcl
resource "random_password" "db_password" {
  length  = 32
  special = true
  override_special = "!#$%&*()-_=+[]{}<>:?"
}
```


### Issue #3: Errors resulting from container builds

**The symptom:** Tasks starting but immediately exiting with `exit code 255` and `gunicorn: exec format error`.

**What I learned:** Building Docker images on an M3 Mac creates ARM64 binaries. ECS Fargate defaults to x86_64 architecture. The mismatch causes exec format errors.

**The fix:** Force x86_64 builds in multiple places:

```dockerfile
FROM --platform=linux/amd64 python:3.11-slim
```

```bash
docker build --platform linux/amd64 -t ecs-todo-api:latest .
```

```hcl
# In ECS task definition
runtime_platform {
  operating_system_family = "LINUX"
  cpu_architecture        = "X86_64"
}
```


### Issue #4: It ain't a real project unless DNS fails somewhere lol

**The symptom:** Health checks passing, but API calls failing with `could not translate host name "xxx.rds.amazonaws.com:5432" to address`.

**What I learned:** The RDS `endpoint` output includes the port (`:5432`), but PostgreSQL's connection string expects hostname and port separately. When you pass `hostname:port` as the hostname, DNS resolution fails.

**The fix:** Use `db_address` instead of `db_endpoint`:

```hcl
environment_vars = {
  DB_HOST = module.rds.db_address  # Hostname only
  DB_PORT = "5432"                 # Port separate
  # ...
}
```


### Issue #5: Code oopsies

**The symptom:** API returning `relation "todos" does not exist` even though init_db() was in the code.

**What I learned:** Code in Python's `if __name__ == '__main__'` block only executes when running the script directly. When Gunicorn imports the module, that block never runs.

**The fix:** Move initialization to module load:

```python
app = Flask(__name__)
CORS(app)

# Initialize database on startup (runs when module is imported)
try:
    init_db()
    logger.info("Database initialized successfully")
except Exception as e:
    logger.error(f"Failed to initialize database: {str(e)}")
```


### Issue #6: Where's my Docker image?

**The symptom:** Terraform created all infrastructure perfectly, but tasks wouldn't start because no Docker image existed in ECR.

**What I forgot:** Terraform creates infrastructure, but building and pushing Docker images is application deployment - they're separate concerns.

**The fix:** Integrated Docker provider directly into Terraform:

```hcl
resource "docker_image" "app" {
  name = "${aws_ecr_repository.main.repository_url}:latest"
  
  build {
    context    = "${path.root}/../app"
    dockerfile = "Dockerfile"
    platform   = "linux/amd64"
  }

  triggers = {
    dockerfile_hash = filemd5("${path.root}/../app/Dockerfile")
    app_hash        = filemd5("${path.root}/../app/app.py")
  }
}

resource "docker_registry_image" "app" {
  name = docker_image.app.name
}
```


### Issue #7: Target group health check timing

**The symptom:** Tasks starting successfully but ALB returning 503 for several minutes.

**What I learned:** Default health check intervals (30 seconds) combined with default healthy threshold (5 checks) means it takes 2.5 minutes for a target to become healthy, even if the app is ready in 10 seconds.

**The optimization:** Tuned health check settings:

```hcl
health_check {
  enabled             = true
  path                = "/health"
  healthy_threshold   = 2  # Down from 5
  unhealthy_threshold = 3
  timeout             = 5
  interval            = 15  # Down from 30
  matcher             = "200"
}
```

**Result:** Healthy targets in 30 seconds instead of 150 seconds. Much better deployment experience.

## Wins: What Works

After all the debugging, here's what I ended up with:

**Complete automation:**
```bash
terraform apply -var-file=environments/dev.tfvars
# 10 minutes later: fully deployed, working API
```

**Full CRUD operations:**
```bash
# Health check
$ curl $ALB_URL/health
{
  "status": "healthy",
  "environment": "dev",
  "version": "1.0.0"
}

# Create TODO
$ curl -X POST $ALB_URL/api/todos \
  -H "Content-Type: application/json" \
  -d '{"title":"Deploy to production","completed":false}'

# Response time: 200-300ms consistently
```

**Auto-scaling:**
- CPU-based scaling: Target 70% utilization
- Memory-based scaling: Target 80% utilization
- Scale from 1 to 10 tasks automatically

**Monitoring:**
- CloudWatch dashboard with ECS, ALB, and RDS metrics
- Alarms for CPU, memory, and database connections
- Centralized logging with structured output

## The Numbers: How much is this gonna cost?

One thing tutorials and blogs almost never talk about: **cost**. Here's the breakdown:

**Monthly costs (dev environment):**
- ECS Fargate (1 task, 0.25 vCPU, 0.5GB): ~$12
- RDS db.t4g.micro, 20GB: ~$15
- Application Load Balancer: ~$20
- NAT Gateways (3 for HA): ~$105
- Data transfer & CloudWatch: ~$8
- **Total: ~$160/month**

**Cost optimization pro tips:**
- NAT Gateways are expensive - consider using 1 instead of 3 for dev
- Fargate pricing is predictable - no surprise costs from scaling
- RDS automated backups add minimal cost
- Scaling to 0 tasks saves ~$12/month when not demoing

For a production environment with 2+ tasks and db.t4g.small, budget ~$250/month.

## Some reminders when building your projects

### 1. Tutorial shortfalls are too common
Following tutorials teaches you syntax. Debugging production issues teaches you engineering.

### 2. Documentation lies (sometimes)
Managed policies don't always work. Default settings aren't always right. Always verify.

### 3. "Works on my machine" still gets you sometimes
ARM vs x86, WSGI vs direct Python, development vs production - the details matter.

### 4. Automation is always worth the time investment
Spending extra time integrating Docker builds into Terraform saved hours on every iteration.

### 5. Real problems make better stories
Every issue I debugged became an interview talking point. That's more valuable than smooth deployments.

## Key Takeaways: If You're Building Your Own Portfolio

### Do:
-  Build something you can deploy and demo
-  Document every issue you solve
-  Use production patterns, not dev shortcuts
-  Calculate and understand costs
-  Make it reproducible with automation

### Don't:
-  Just follow tutorials without understanding why
-  Build something you can't explain under pressure
-  Ignore cost implications
-  Skip documentation "for now"
-  Build locally-only infrastructure

### Remember:
The debugging is the learning. The mistakes are the stories. The struggle is what makes it authentic.

## Future enhancements

Potential improvements for production readiness:

**Short term:**
- HTTPS with ACM certificate
- Custom domain with Route53
- Blue/green deployments with CodeDeploy

**Medium term:**
- WAF for additional security
- Database backup/restore procedures
- Integration tests in CI/CD

**Long term:**
- Multi-region deployment
- Disaster recovery testing
- Performance optimization and load testing


## Final Thoughts

After this project, I'm not just "current" with DevOps practices - I have concrete examples of modern infrastructure, real debugging wins, and production thinking.

When a recruiter asks "what have you been working on?", I can send them a live URL, a GitHub repo with comprehensive documentation, and explain seven different production issues I solved.

That's worth more than any certification or tutorial completion certificate.

**If you're rebooting your DevOps career, don't just update your resume. Build something real. Debug something hard. Document it well.**

---

## Resources

**GitHub Repository**: [github.com/treehousepnw/ecs-todo-api-project](https://github.com/treehousepnw/ecs-todo-api-project) 
**Live Demo**: Available upon request (may be scaled down to save costs)  
**Tech Stack**: AWS (ECS, RDS, VPC, ALB, ECR), Terraform, Docker, Python/Flask, PostgreSQL

**Connect with me:**
- Portfolio: [trenigma.dev](https://trenigma.dev)
- LinkedIn: [linkedin.com/in/trenigma](https://linkedin.com/in/trenigma)
- Email: derek.ogletree@gmail.com

---

*Built this project? Have questions about the debugging process? Found a better approach? I'd love to hear from you. Shoot me an email or reach out on LinkedIn.*

**Tags**: #DevOps #AWS #Terraform #ECS #Docker #Infrastructure #Career #SRE #CloudEngineering