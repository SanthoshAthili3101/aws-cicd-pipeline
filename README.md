.NET Web API on AWS with CodeCommit, CodeBuild, and CodeDeploy
A production-ready .NET Web API project wired to an AWS-native CI/CD pipeline. Source is hosted on CodeCommit, builds are executed by CodeBuild, and deployments are orchestrated by CodeDeploy through CodePipeline.

Features
.NET 8 Web API with RESTful endpoints and Swagger.

CI/CD via AWS Developer Tools: CodeCommit → CodeBuild → CodeDeploy → EC2/On‑premises.

Artifact packaging and versioned deployments with automatic rollback on failure.

Environment-specific configuration and zero manual steps after commit to main.

Architecture
Source: CodeCommit repository (main branch triggers).

Build: CodeBuild project executes buildspec.yml to restore, test, publish, and package artifacts.

Deploy: CodeDeploy Application + Deployment Group deploys to EC2 using AppSpec hooks.

Orchestration: CodePipeline connects stages, artifacts, and triggers.

Prerequisites
AWS account with IAM permissions to provision CodeCommit, CodeBuild, CodeDeploy, CodePipeline, S3, and EC2.

An EC2 instance (or fleet) with the CodeDeploy agent installed and reachable by CodeDeploy.

S3 artifact bucket for CodePipeline artifacts.

.NET 8 SDK locally for development; .NET 8 runtime on target instances if not self-contained publish.

Repository Structure
src/MyApi/MyApi.csproj — .NET Web API project.

buildspec.yml — Build instructions for CodeBuild.

appspec.yml — CodeDeploy deployment spec.

scripts/ — Lifecycle hook scripts for CodeDeploy (BeforeInstall, AfterInstall, ApplicationStart, ValidateService).

pipeline/ — Optional IaC templates for pipeline resources.

README.md — This file.

buildspec.yml (example)
version: 0.2
phases:
install:
runtime-versions:
dotnet: 8.0
pre_build:
commands:
- dotnet --info
- dotnet restore src/MyApi/MyApi.csproj
- dotnet test --no-build --verbosity normal
build:
commands:
- dotnet publish src/MyApi/MyApi.csproj -c Release -o publish
- mkdir -p artifact
- cp -r publish/* artifact/
- cp appspec.yml artifact/
- cp -r scripts artifact/scripts
artifacts:
files:
- '**/*'
base-directory: artifact

Notes:

Uses .NET 8 managed runtime in CodeBuild.

Publishes to artifact/ and includes appspec.yml + scripts for CodeDeploy.

Add environment variables in the CodeBuild project if needed.

appspec.yml (EC2/On-premises example)
version: 0.0
os: linux
files:

source: /
destination: /var/www/myapi
permissions:

object: /var/www/myapi
mode: 755
type:

file
hooks:
BeforeInstall:

location: scripts/before_install.sh
timeout: 300
runas: root
AfterInstall:

location: scripts/after_install.sh
timeout: 300
runas: root
ApplicationStart:

location: scripts/application_start.sh
timeout: 300
runas: root
ValidateService:

location: scripts/validate_service.sh
timeout: 300
runas: root

Example lifecycle scripts
scripts/before_install.sh
#!/usr/bin/env bash
set -e
systemctl stop myapi.service || true
mkdir -p /var/www/myapi

scripts/after_install.sh
#!/usr/bin/env bash
set -e
chown -R ec2-user:ec2-user /var/www/myapi

scripts/application_start.sh
#!/usr/bin/env bash
set -e
cat >/etc/systemd/system/myapi.service <<'UNIT'
[Unit]
Description=MyApi .NET Web API
After=network.target

[Service]
WorkingDirectory=/var/www/myapi
ExecStart=/usr/bin/dotnet /var/www/myapi/MyApi.dll --urls "http://*:5000"
Restart=always
RestartSec=5
User=ec2-user
Environment=ASPNETCORE_ENVIRONMENT=Production

[Install]
WantedBy=multi-user.target
UNIT

systemctl daemon-reload
systemctl enable myapi.service
systemctl restart myapi.service

scripts/validate_service.sh
#!/usr/bin/env bash
set -e
curl -sf http://localhost:5000/health || curl -sf http://localhost:5000/swagger || exit 1

Tip:

Prefer a dedicated /health endpoint over /swagger for health checks.

Adjust user, paths, and port to match instance setup.

Local Development
Restore: dotnet restore src/MyApi/MyApi.csproj

Run: dotnet run --project src/MyApi/MyApi.csproj

Swagger: http://localhost:5000/swagger

Recommended: appsettings.Development.json for local config.

CI/CD Setup Steps (high level)
CodeCommit

Create a repository and push this project (main branch).

Configure Git credential helper or SSH for CodeCommit.

S3 Artifact Bucket

Create an S3 bucket for pipeline artifacts in the same region.

CodeDeploy

Install CodeDeploy agent on EC2:

Amazon Linux 2: sudo yum install -y ruby; install agent via official script/package.

Create CodeDeploy Application (EC2/On‑prem) and Deployment Group targeting your instances via tags or ASG.

CodeBuild

Create a build project with .NET 8 managed image.

Use buildspec.yml from the repo.

Grant access to S3 artifact bucket.

CodePipeline

Source: CodeCommit (main).

Build: CodeBuild project above.

Deploy: CodeDeploy Application + Deployment Group.

Provide the S3 artifact bucket and service roles as prompted.

Configuration
Port: Default 5000; change via systemd ExecStart or ASPNETCORE_URLS.

Environment: Set ASPNETCORE_ENVIRONMENT via systemd or CodeDeploy hooks.

Secrets: Use AWS Systems Manager Parameter Store or Secrets Manager in app configuration.

Security Best Practices
Least-privilege IAM roles for CodeBuild, CodeDeploy, and CodePipeline.

Restrict inbound Security Group rules; front the API with ALB and TLS on 443.

Avoid storing secrets in plaintext; prefer Parameter Store/Secrets Manager.

Enable CloudWatch Logs or a log forwarder for audit and diagnostics.

Observability
Add Serilog or built-in logging to CloudWatch Logs.

Expose /health for liveness/readiness; integrate with ALB health checks.

Emit metrics for key business and system signals.

Troubleshooting
CodeBuild failing: check buildspec phases and IAM for S3/artifact access.

CodeDeploy “Agent not found”: verify agent installation and that instance is in the correct Deployment Group (tags/ASG).

Deployment fails at ValidateService: inspect instance logs (journalctl -u myapi -e), curl localhost:5000, review scripts.

Port conflicts: update port or stop previous processes; confirm Security Group rules.

Roadmap
Blue/Green or Rolling deployments in CodeDeploy with ALB traffic shifting.

Infrastructure as Code using CloudFormation/Terraform for repeatable environments.

Multi-environment pipelines (dev/stage/prod) with approval gates.

Canary validations and automated rollback signals.
