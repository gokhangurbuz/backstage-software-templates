# ${{values.app_name | capitalize}}

Welcome to the technical documentation for **${{values.app_name}}**.

## Overview

${{values.app_name}} is a Python-based microservice that provides system information and health check endpoints. This service is deployed in the **${{values.app_env}}** environment and is part of the Roofstacks platform ecosystem.

## Quick Links

- [Getting Started](getting-started/overview.md) - Start here if you're new
- [API Reference](api/overview.md) - Complete API documentation
- [Deployment Guide](deployment/kubernetes.md) - Deploy to production
- [Troubleshooting](operations/troubleshooting.md) - Common issues and solutions

## Key Features

- ✅ RESTful API endpoints for system monitoring
- ✅ Health check capabilities for orchestration tools
- ✅ Environment-aware configuration
- ✅ Lightweight and containerized deployment
- ✅ Production-ready with monitoring and logging

## Service Information

| Property | Value |
|----------|-------|
| **Service Name** | ${{values.app_name}} |
| **Environment** | ${{values.app_env}} |
| **Base URL** | `https://${{values.app_name}}-${{values.app_env}}.roofstacks.com` |
| **Team** | ${{values.team_name | default("Platform Team")}} |

## Quick Access

Test the service endpoints:

- **Health Check**: [`/api/v1/healthz`](https://${{values.app_name}}-${{values.app_env}}.roofstacks.com/api/v1/healthz)
- **System Info**: [`/api/v1/info`](https://${{values.app_name}}-${{values.app_env}}.roofstacks.com/api/v1/info)

## Support

Need help? Reach out to:

- **Slack Channel**: #${{values.app_name}}-support
- **Team**: ${{values.team_name | default("Platform Team")}}
- **Repository**: [${{values.repo_url}}](${{values.repo_url}})