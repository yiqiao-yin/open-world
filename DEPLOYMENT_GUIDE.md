# World Monitor — Cloud Deployment Guide

A step-by-step guide for deploying World Monitor to **Azure** or **AWS**.
This document is written to be unambiguous enough to serve as a system prompt
for an AI engineer or a runbook for a human operator.

---

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Prerequisites](#prerequisites)
- [Step 0 — Build the Frontend](#step-0--build-the-frontend)
- [Step 1 — Environment Variables](#step-1--environment-variables)
- [Option A — Deploy to Azure](#option-a--deploy-to-azure)
  - [A1. Static Frontend → Azure Static Web Apps](#a1-static-frontend--azure-static-web-apps)
  - [A2. API Functions → Azure Functions](#a2-api-functions--azure-functions)
  - [A3. Middleware → Azure Front Door](#a3-middleware--azure-front-door)
  - [A4. Cron Job → Azure Timer Trigger](#a4-cron-job--azure-timer-trigger)
  - [A5. Relay Server → Azure App Service](#a5-relay-server--azure-app-service)
  - [A6. Verify Azure Deployment](#a6-verify-azure-deployment)
- [Option B — Deploy to AWS](#option-b--deploy-to-aws)
  - [B1. Static Frontend → S3 + CloudFront](#b1-static-frontend--s3--cloudfront)
  - [B2. API Functions → API Gateway + Lambda](#b2-api-functions--api-gateway--lambda)
  - [B3. Middleware → CloudFront Functions](#b3-middleware--cloudfront-functions)
  - [B4. Cron Job → EventBridge + Lambda](#b4-cron-job--eventbridge--lambda)
  - [B5. Relay Server → ECS Fargate](#b5-relay-server--ecs-fargate)
  - [B6. Verify AWS Deployment](#b6-verify-aws-deployment)
- [Adapter Layer — Porting Vercel Edge Functions](#adapter-layer--porting-vercel-edge-functions)
- [DNS & Custom Domain Setup](#dns--custom-domain-setup)
- [Monitoring & Observability](#monitoring--observability)
- [Cost Estimates](#cost-estimates)
- [Troubleshooting](#troubleshooting)

---

## Architecture Overview

World Monitor has **three independently deployable components**:

```
┌─────────────────────────────────────────────────────────────────┐
│  1. STATIC FRONTEND (Vite SPA)                                  │
│     Build output: dist/                                         │
│     Content: HTML, JS, CSS, WASM, images (brotli-compressed)   │
│     Size: ~15-25 MB built                                       │
│     Framework: Vite + TypeScript (vanilla, no React/Vue)        │
├─────────────────────────────────────────────────────────────────┤
│  2. API FUNCTIONS (46 serverless endpoints)                     │
│     Source: api/ + server/                                      │
│     Runtime: Edge-compatible (Web Request/Response APIs)        │
│     Auth: CORS allowlist + optional API key + rate limiting     │
│     Patterns:                                                   │
│       - RPC gateway (api/*/v1/[rpc].ts → server/gateway.ts)    │
│       - Standalone handlers (api/rss-proxy.js, api/opensky.js) │
│       - Cron handler (api/cron/warm-aviation-cache.ts)          │
├─────────────────────────────────────────────────────────────────┤
│  3. RELAY SERVER (long-running Node.js process)                 │
│     Entry: node scripts/ais-relay.cjs                           │
│     Port: 3004 (configurable via PORT env var)                  │
│     Protocols: HTTP + WebSocket                                 │
│     Handles: AIS vessels, OpenSky aircraft, RSS proxy,          │
│              Telegram OSINT, OREF alerts                        │
│     Memory: ~512 MB recommended                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Current vs Target Mapping

| Component | Vercel (Current) | Azure (Target) | AWS (Target) |
|---|---|---|---|
| Static frontend | Vercel Edge CDN | Azure Static Web Apps | S3 + CloudFront |
| 46 API functions | Vercel Serverless/Edge | Azure Functions (Node.js 20) | API Gateway + Lambda |
| Middleware | middleware.ts | Azure Front Door rules | CloudFront Functions |
| Cron (aviation cache) | Vercel Crons | Azure Timer Trigger | EventBridge + Lambda |
| Relay server | Railway (Node.js) | Azure App Service (always-on) | ECS Fargate |
| Cache | Upstash Redis (REST) | Upstash Redis (keep as-is) | Upstash Redis (keep as-is) |

> **Note on Upstash Redis:** The codebase uses Upstash Redis via its REST API
> (not TCP). This means it works from any cloud provider without changes.
> You do NOT need to migrate to Azure Cache for Redis or Amazon ElastiCache.

---

## Prerequisites

Before starting, ensure you have:

```bash
# Node.js 20+ and npm
node --version   # v20.x or v22.x
npm --version    # 10.x+

# Azure CLI (for Azure deployment)
az --version     # 2.60+
az login

# AWS CLI v2 (for AWS deployment)
aws --version    # aws-cli/2.x
aws configure    # Set up credentials + default region

# GitHub CLI (for repo access)
gh auth status

# Project dependencies
cd /path/to/worldmonitor
npm install
```

### Required CLI Tools by Platform

**Azure:**
```bash
# Install Azure CLI
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash

# Install Azure Functions Core Tools
npm install -g azure-functions-core-tools@4 --unsafe-perm true

# Install Azure Static Web Apps CLI
npm install -g @azure/static-web-apps-cli
```

**AWS:**
```bash
# Install AWS CLI v2
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip && sudo ./aws/install

# Install AWS CDK (optional, for IaC)
npm install -g aws-cdk

# Install AWS SAM CLI (for Lambda local testing)
pip install aws-sam-cli
```

---

## Step 0 — Build the Frontend

The frontend build is identical regardless of target cloud. Run from project root:

```bash
# Install dependencies
npm install

# Build the default (full) variant
npm run build

# Or build a specific variant
npm run build:tech      # Tech Monitor
npm run build:finance   # Finance Monitor
npm run build:happy     # Happy Monitor
```

**Output:** `dist/` directory containing:
- `index.html` — SPA entry point
- `assets/` — hashed JS/CSS bundles (cache-immutable)
- `*.br` — brotli-precompressed copies of all assets
- `sw.js` — service worker for PWA/offline
- `manifest.webmanifest` — PWA manifest
- `offline.html` — offline fallback page
- `favico/` — favicon and OG image assets

Verify the build:
```bash
ls -la dist/
# Should see index.html, assets/, sw.js, manifest.webmanifest, offline.html
du -sh dist/
# Expect ~15-25 MB
```

---

## Step 1 — Environment Variables

Copy the template and fill in your keys:

```bash
cp .env.example .env.production
```

**Minimum viable deployment** (dashboard works with reduced features):
```bash
# .env.production — minimum
UPSTASH_REDIS_REST_URL=https://your-redis.upstash.io
UPSTASH_REDIS_REST_TOKEN=AX...your-token
```

**Recommended deployment** (most features enabled):
```bash
# .env.production — recommended
UPSTASH_REDIS_REST_URL=https://your-redis.upstash.io
UPSTASH_REDIS_REST_TOKEN=AX...your-token
GROQ_API_KEY=gsk_...
FINNHUB_API_KEY=...
FRED_API_KEY=...
EIA_API_KEY=...
NASA_FIRMS_API_KEY=...
ACLED_ACCESS_TOKEN=...
CLOUDFLARE_API_TOKEN=...
```

See [DATA_SOURCES.md](./DATA_SOURCES.md) for the full list with registration links.

---

## Option A — Deploy to Azure

### A1. Static Frontend → Azure Static Web Apps

Azure Static Web Apps serves the `dist/` directory with global CDN, free SSL,
and custom domain support.

```bash
# 1. Create a resource group (if you don't have one)
az group create \
  --name rg-worldmonitor \
  --location eastus2

# 2. Create the Static Web App
az staticwebapp create \
  --name swa-worldmonitor \
  --resource-group rg-worldmonitor \
  --location eastus2 \
  --sku Free

# 3. Get the deployment token
DEPLOY_TOKEN=$(az staticwebapp secrets list \
  --name swa-worldmonitor \
  --resource-group rg-worldmonitor \
  --query "properties.apiKey" -o tsv)

# 4. Deploy the built frontend using the SWA CLI
npx @azure/static-web-apps-cli deploy ./dist \
  --deployment-token "$DEPLOY_TOKEN" \
  --env production
```

**Configure cache headers** by creating `staticwebapp.config.json` in the `dist/` directory:

```json
{
  "routes": [
    { "route": "/assets/*", "headers": { "Cache-Control": "public, max-age=31536000, immutable" } },
    { "route": "/sw.js", "headers": { "Cache-Control": "public, max-age=0, must-revalidate" } },
    { "route": "/manifest.webmanifest", "headers": { "Cache-Control": "public, max-age=86400" } },
    { "route": "/offline.html", "headers": { "Cache-Control": "public, max-age=86400" } },
    { "route": "/favico/*", "headers": { "Cache-Control": "public, max-age=604800" } }
  ],
  "navigationFallback": {
    "rewrite": "/index.html",
    "exclude": ["/assets/*", "/favico/*", "/api/*"]
  },
  "globalHeaders": {
    "X-Content-Type-Options": "nosniff",
    "X-Frame-Options": "SAMEORIGIN",
    "Strict-Transport-Security": "max-age=63072000; includeSubDomains; preload",
    "Referrer-Policy": "strict-origin-when-cross-origin"
  }
}
```

Copy it into dist before deploying:
```bash
cp staticwebapp.config.json dist/
```

---

### A2. API Functions → Azure Functions

The 46 API endpoints need to be ported from Vercel's edge function format
to Azure Functions. This requires an adapter layer (see
[Adapter Layer](#adapter-layer--porting-vercel-edge-functions) section below).

```bash
# 1. Create a Storage Account (required by Azure Functions)
az storage account create \
  --name stworldmonitor \
  --resource-group rg-worldmonitor \
  --location eastus2 \
  --sku Standard_LRS

# 2. Create the Function App (Node.js 20, Linux, Consumption plan)
az functionapp create \
  --name func-worldmonitor \
  --resource-group rg-worldmonitor \
  --storage-account stworldmonitor \
  --consumption-plan-location eastus2 \
  --runtime node \
  --runtime-version 20 \
  --os-type Linux \
  --functions-version 4

# 3. Set environment variables (from your .env.production)
az functionapp config appsettings set \
  --name func-worldmonitor \
  --resource-group rg-worldmonitor \
  --settings \
    UPSTASH_REDIS_REST_URL="https://your-redis.upstash.io" \
    UPSTASH_REDIS_REST_TOKEN="AX...your-token" \
    GROQ_API_KEY="gsk_..." \
    FINNHUB_API_KEY="..." \
    FRED_API_KEY="..." \
    EIA_API_KEY="..." \
    NASA_FIRMS_API_KEY="..." \
    ACLED_ACCESS_TOKEN="..." \
    CLOUDFLARE_API_TOKEN="..." \
    CRON_SECRET="$(openssl rand -hex 32)"

# 4. Create the Azure Functions project structure
#    (See Adapter Layer section for the adapter code)
mkdir -p azure-functions
cd azure-functions
func init --worker-runtime node --language typescript --model V4

# 5. Deploy the functions
func azure functionapp publish func-worldmonitor

# 6. Verify deployment
az functionapp function list \
  --name func-worldmonitor \
  --resource-group rg-worldmonitor \
  --output table
```

**Azure Functions `host.json`** — place in `azure-functions/`:
```json
{
  "version": "2.0",
  "extensions": {
    "http": {
      "routePrefix": "api",
      "maxConcurrentRequests": 200,
      "dynamicThrottlesEnabled": false
    }
  },
  "functionTimeout": "00:00:30",
  "logging": {
    "logLevel": {
      "default": "Warning",
      "Function": "Information"
    }
  }
}
```

---

### A3. Middleware → Azure Front Door

The Vercel middleware handles bot filtering and social-preview OG responses.
On Azure, replicate this with Azure Front Door rules.

```bash
# 1. Create Azure Front Door profile
az afd profile create \
  --profile-name afd-worldmonitor \
  --resource-group rg-worldmonitor \
  --sku Standard_AzureFrontDoor

# 2. Create endpoint
az afd endpoint create \
  --endpoint-name worldmonitor \
  --profile-name afd-worldmonitor \
  --resource-group rg-worldmonitor

# 3. Create origin group for the static site
az afd origin-group create \
  --origin-group-name og-static \
  --profile-name afd-worldmonitor \
  --resource-group rg-worldmonitor \
  --probe-request-type GET \
  --probe-protocol Https \
  --probe-interval-in-seconds 60 \
  --probe-path "/" \
  --sample-size 4 \
  --successful-samples-required 3

# 4. Add the Static Web App as an origin
SWA_HOSTNAME=$(az staticwebapp show \
  --name swa-worldmonitor \
  --resource-group rg-worldmonitor \
  --query "defaultHostname" -o tsv)

az afd origin create \
  --origin-name origin-swa \
  --origin-group-name og-static \
  --profile-name afd-worldmonitor \
  --resource-group rg-worldmonitor \
  --host-name "$SWA_HOSTNAME" \
  --origin-host-header "$SWA_HOSTNAME" \
  --http-port 80 \
  --https-port 443 \
  --priority 1

# 5. Create origin group for the API Function App
az afd origin-group create \
  --origin-group-name og-api \
  --profile-name afd-worldmonitor \
  --resource-group rg-worldmonitor \
  --probe-request-type GET \
  --probe-protocol Https \
  --probe-interval-in-seconds 30 \
  --probe-path "/api/version" \
  --sample-size 4 \
  --successful-samples-required 3

FUNC_HOSTNAME=$(az functionapp show \
  --name func-worldmonitor \
  --resource-group rg-worldmonitor \
  --query "defaultHostName" -o tsv)

az afd origin create \
  --origin-name origin-api \
  --origin-group-name og-api \
  --profile-name afd-worldmonitor \
  --resource-group rg-worldmonitor \
  --host-name "$FUNC_HOSTNAME" \
  --origin-host-header "$FUNC_HOSTNAME" \
  --http-port 80 \
  --https-port 443 \
  --priority 1

# 6. Create route: /api/* → Function App
az afd route create \
  --route-name route-api \
  --endpoint-name worldmonitor \
  --profile-name afd-worldmonitor \
  --resource-group rg-worldmonitor \
  --origin-group og-api \
  --patterns-to-match "/api/*" \
  --supported-protocols Https \
  --forwarding-protocol HttpsOnly \
  --https-redirect Enabled

# 7. Create route: /* → Static Web App (default)
az afd route create \
  --route-name route-static \
  --endpoint-name worldmonitor \
  --profile-name afd-worldmonitor \
  --resource-group rg-worldmonitor \
  --origin-group og-static \
  --patterns-to-match "/*" \
  --supported-protocols Https \
  --forwarding-protocol HttpsOnly \
  --https-redirect Enabled
```

**Bot filtering rule set** (replaces middleware.ts bot blocking):

```bash
# Create a WAF policy with bot filtering
az afd security-policy create \
  --profile-name afd-worldmonitor \
  --resource-group rg-worldmonitor \
  --security-policy-name waf-bots \
  --domains worldmonitor \
  --waf-policy "/subscriptions/<SUB_ID>/resourceGroups/rg-worldmonitor/providers/Microsoft.Network/FrontDoorWebApplicationFirewallPolicies/waf-worldmonitor"
```

> **Note:** For full parity with the Vercel middleware (social preview OG
> responses for variant subdomains), you can implement a small Azure Function
> at `/api/og-preview` that returns the OG HTML, and configure Front Door
> to route social bot User-Agents to it via Rules Engine.

---

### A4. Cron Job → Azure Timer Trigger

The aviation cache warmer runs every 2 hours. On Azure, this becomes a
Timer Trigger function.

In your `azure-functions/` project, create `src/functions/warmAviationCache.ts`:

```typescript
import { app, InvocationContext, Timer } from "@azure/functions";

// Import the adapted handler from the adapter layer
import { warmAviationCache } from "../adapters/cron-adapter";

app.timer("warmAviationCache", {
  // Every 2 hours (same as Vercel: "0 */2 * * *")
  schedule: "0 */2 * * *",
  handler: async (myTimer: Timer, context: InvocationContext) => {
    context.log("Aviation cache warmer triggered at:", new Date().toISOString());
    try {
      await warmAviationCache();
      context.log("Aviation cache warmer completed successfully");
    } catch (err) {
      context.error("Aviation cache warmer failed:", err);
    }
  },
});
```

Deploy with the rest of the functions:
```bash
func azure functionapp publish func-worldmonitor
```

---

### A5. Relay Server → Azure App Service

The relay server is a long-running Node.js process that maintains WebSocket
connections. It cannot be serverless — it needs an always-on compute instance.

```bash
# 1. Create an App Service Plan (B1 = Basic tier, always-on capable)
az appservice plan create \
  --name plan-worldmonitor-relay \
  --resource-group rg-worldmonitor \
  --location eastus2 \
  --sku B1 \
  --is-linux

# 2. Create the Web App
az webapp create \
  --name app-worldmonitor-relay \
  --resource-group rg-worldmonitor \
  --plan plan-worldmonitor-relay \
  --runtime "NODE:20-lts"

# 3. Enable Always On (critical — prevents idle shutdown)
az webapp config set \
  --name app-worldmonitor-relay \
  --resource-group rg-worldmonitor \
  --always-on true

# 4. Enable WebSocket support
az webapp config set \
  --name app-worldmonitor-relay \
  --resource-group rg-worldmonitor \
  --web-sockets-enabled true

# 5. Set the startup command
az webapp config set \
  --name app-worldmonitor-relay \
  --resource-group rg-worldmonitor \
  --startup-file "node scripts/ais-relay.cjs"

# 6. Configure environment variables
RELAY_SECRET=$(openssl rand -hex 32)

az webapp config appsettings set \
  --name app-worldmonitor-relay \
  --resource-group rg-worldmonitor \
  --settings \
    PORT="3004" \
    RELAY_SHARED_SECRET="$RELAY_SECRET" \
    RELAY_AUTH_HEADER="x-relay-key" \
    ALLOW_UNAUTHENTICATED_RELAY="false" \
    UPSTASH_REDIS_REST_URL="https://your-redis.upstash.io" \
    UPSTASH_REDIS_REST_TOKEN="AX...your-token" \
    AISSTREAM_API_KEY="..." \
    OPENSKY_CLIENT_ID="..." \
    OPENSKY_CLIENT_SECRET="..."

# 7. Deploy the code via ZIP deploy
#    First, create a deployment package with only what the relay needs:
mkdir -p /tmp/relay-deploy
cp package.json package-lock.json /tmp/relay-deploy/
cp -r scripts/ /tmp/relay-deploy/
cp -r api/ /tmp/relay-deploy/       # Needed for RSS proxy handler references
cp -r server/ /tmp/relay-deploy/    # Shared utilities
cp -r data/ /tmp/relay-deploy/      # Data files if any
cd /tmp/relay-deploy
npm install --omit=dev
zip -r ../relay.zip .

az webapp deployment source config-zip \
  --name app-worldmonitor-relay \
  --resource-group rg-worldmonitor \
  --src /tmp/relay.zip

# 8. Now update the Function App to point to this relay
RELAY_URL="https://app-worldmonitor-relay.azurewebsites.net"

az functionapp config appsettings set \
  --name func-worldmonitor \
  --resource-group rg-worldmonitor \
  --settings \
    WS_RELAY_URL="$RELAY_URL" \
    RELAY_SHARED_SECRET="$RELAY_SECRET" \
    RELAY_AUTH_HEADER="x-relay-key"

# 9. Verify the relay is running
curl -s "https://app-worldmonitor-relay.azurewebsites.net/health"
# Should return: {"ok":true,...}
```

---

### A6. Verify Azure Deployment

```bash
# Check Static Web App
SWA_URL=$(az staticwebapp show \
  --name swa-worldmonitor \
  --resource-group rg-worldmonitor \
  --query "defaultHostname" -o tsv)
echo "Frontend: https://$SWA_URL"
curl -s -o /dev/null -w "%{http_code}" "https://$SWA_URL"
# Expect: 200

# Check API Functions
FUNC_URL=$(az functionapp show \
  --name func-worldmonitor \
  --resource-group rg-worldmonitor \
  --query "defaultHostName" -o tsv)
echo "API: https://$FUNC_URL"
curl -s "https://$FUNC_URL/api/version"
# Expect: {"version":"2.5.23",...}

# Check Relay Server
curl -s "https://app-worldmonitor-relay.azurewebsites.net/health"
# Expect: {"ok":true,...}

# Check an API endpoint that requires Redis
curl -s "https://$FUNC_URL/api/seismology/v1/list-earthquakes" | head -c 200
# Expect: JSON with earthquake data

# Check Front Door endpoint (if configured)
AFD_ENDPOINT=$(az afd endpoint show \
  --endpoint-name worldmonitor \
  --profile-name afd-worldmonitor \
  --resource-group rg-worldmonitor \
  --query "hostName" -o tsv)
echo "CDN: https://$AFD_ENDPOINT"
```

---

## Option B — Deploy to AWS

### B1. Static Frontend → S3 + CloudFront

```bash
# 1. Create the S3 bucket
BUCKET_NAME="worldmonitor-frontend-$(aws sts get-caller-identity --query Account --output text)"

aws s3 mb "s3://$BUCKET_NAME" --region us-east-1

# 2. Configure the bucket for static website hosting
aws s3 website "s3://$BUCKET_NAME" \
  --index-document index.html \
  --error-document index.html

# 3. Upload the built frontend
aws s3 sync dist/ "s3://$BUCKET_NAME/" \
  --delete \
  --cache-control "no-cache, no-store, must-revalidate" \
  --exclude "assets/*" \
  --exclude "favico/*" \
  --exclude "*.br"

# Upload assets with long-lived cache
aws s3 sync dist/assets/ "s3://$BUCKET_NAME/assets/" \
  --cache-control "public, max-age=31536000, immutable" \
  --exclude "*.br"

# Upload favicons with 7-day cache
aws s3 sync dist/favico/ "s3://$BUCKET_NAME/favico/" \
  --cache-control "public, max-age=604800"

# Upload service worker with no-cache
aws s3 cp dist/sw.js "s3://$BUCKET_NAME/sw.js" \
  --cache-control "public, max-age=0, must-revalidate"

# Upload PWA manifest
aws s3 cp dist/manifest.webmanifest "s3://$BUCKET_NAME/manifest.webmanifest" \
  --cache-control "public, max-age=86400"

# 4. Create CloudFront Origin Access Control
OAC_ID=$(aws cloudfront create-origin-access-control \
  --origin-access-control-config '{
    "Name": "worldmonitor-oac",
    "Description": "OAC for World Monitor S3",
    "SigningProtocol": "sigv4",
    "SigningBehavior": "always",
    "OriginAccessControlOriginType": "s3"
  }' \
  --query "OriginAccessControl.Id" --output text)

# 5. Create CloudFront distribution
#    Save the following as cloudfront-config.json:
cat > /tmp/cloudfront-config.json << 'CFEOF'
{
  "CallerReference": "worldmonitor-$(date +%s)",
  "Comment": "World Monitor Frontend",
  "DefaultCacheBehavior": {
    "TargetOriginId": "s3-worldmonitor",
    "ViewerProtocolPolicy": "redirect-to-https",
    "CachePolicyId": "658327ea-f89d-4fab-a63d-7e88639e58f6",
    "Compress": true,
    "AllowedMethods": ["GET", "HEAD", "OPTIONS"],
    "CachedMethods": ["GET", "HEAD"]
  },
  "Origins": {
    "Quantity": 1,
    "Items": [
      {
        "Id": "s3-worldmonitor",
        "DomainName": "BUCKET_NAME.s3.us-east-1.amazonaws.com",
        "S3OriginConfig": { "OriginAccessIdentity": "" },
        "OriginAccessControlId": "OAC_ID_PLACEHOLDER"
      }
    ]
  },
  "Enabled": true,
  "DefaultRootObject": "index.html",
  "CustomErrorResponses": {
    "Quantity": 1,
    "Items": [
      {
        "ErrorCode": 404,
        "ResponsePagePath": "/index.html",
        "ResponseCode": "200",
        "ErrorCachingMinTTL": 0
      }
    ]
  },
  "HttpVersion": "http2and3",
  "PriceClass": "PriceClass_100"
}
CFEOF

# Replace placeholders
sed -i "s/BUCKET_NAME/$BUCKET_NAME/g" /tmp/cloudfront-config.json
sed -i "s/OAC_ID_PLACEHOLDER/$OAC_ID/g" /tmp/cloudfront-config.json

DISTRIBUTION_ID=$(aws cloudfront create-distribution \
  --distribution-config file:///tmp/cloudfront-config.json \
  --query "Distribution.Id" --output text)

DISTRIBUTION_DOMAIN=$(aws cloudfront get-distribution \
  --id "$DISTRIBUTION_ID" \
  --query "Distribution.DomainName" --output text)

echo "CloudFront: https://$DISTRIBUTION_DOMAIN"

# 6. Update S3 bucket policy to allow CloudFront OAC access
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

cat > /tmp/bucket-policy.json << BPEOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowCloudFrontOAC",
      "Effect": "Allow",
      "Principal": { "Service": "cloudfront.amazonaws.com" },
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::$BUCKET_NAME/*",
      "Condition": {
        "StringEquals": {
          "AWS:SourceArn": "arn:aws:cloudfront::$ACCOUNT_ID:distribution/$DISTRIBUTION_ID"
        }
      }
    }
  ]
}
BPEOF

aws s3api put-bucket-policy \
  --bucket "$BUCKET_NAME" \
  --policy file:///tmp/bucket-policy.json
```

---

### B2. API Functions → API Gateway + Lambda

Each API endpoint becomes a Lambda function behind API Gateway.
The adapter layer (see [below](#adapter-layer--porting-vercel-edge-functions))
wraps the existing handlers.

```bash
# 1. Create the Lambda execution role
aws iam create-role \
  --role-name worldmonitor-lambda-role \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": { "Service": "lambda.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }]
  }'

aws iam attach-role-policy \
  --role-name worldmonitor-lambda-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

LAMBDA_ROLE_ARN=$(aws iam get-role \
  --role-name worldmonitor-lambda-role \
  --query "Role.Arn" --output text)

# Wait for role propagation
sleep 10

# 2. Package the Lambda function
#    (After creating the adapter layer — see Adapter section)
mkdir -p /tmp/lambda-deploy
cp -r api/ /tmp/lambda-deploy/
cp -r server/ /tmp/lambda-deploy/
cp -r src/generated/ /tmp/lambda-deploy/src/generated/
cp -r data/ /tmp/lambda-deploy/ 2>/dev/null || true
cp package.json package-lock.json /tmp/lambda-deploy/
cp lambda-adapter.mjs /tmp/lambda-deploy/   # Your adapter file (see below)

cd /tmp/lambda-deploy
npm install --omit=dev
zip -r ../lambda-api.zip .

# 3. Create the Lambda function
aws lambda create-function \
  --function-name worldmonitor-api \
  --runtime nodejs20.x \
  --role "$LAMBDA_ROLE_ARN" \
  --handler lambda-adapter.handler \
  --zip-file fileb:///tmp/lambda-api.zip \
  --timeout 30 \
  --memory-size 512 \
  --environment "Variables={
    UPSTASH_REDIS_REST_URL=https://your-redis.upstash.io,
    UPSTASH_REDIS_REST_TOKEN=AX...your-token,
    GROQ_API_KEY=gsk_...,
    FINNHUB_API_KEY=...,
    FRED_API_KEY=...,
    EIA_API_KEY=...,
    NASA_FIRMS_API_KEY=...,
    ACLED_ACCESS_TOKEN=...,
    CLOUDFLARE_API_TOKEN=...
  }"

# 4. Create the HTTP API Gateway
API_ID=$(aws apigatewayv2 create-api \
  --name worldmonitor-api \
  --protocol-type HTTP \
  --cors-configuration '{
    "AllowOrigins": ["*"],
    "AllowMethods": ["GET", "POST", "OPTIONS"],
    "AllowHeaders": ["Content-Type", "Authorization", "X-WorldMonitor-Key"],
    "MaxAge": 86400
  }' \
  --query "ApiId" --output text)

# 5. Create Lambda integration
INTEGRATION_ID=$(aws apigatewayv2 create-integration \
  --api-id "$API_ID" \
  --integration-type AWS_PROXY \
  --integration-uri "arn:aws:lambda:us-east-1:$ACCOUNT_ID:function:worldmonitor-api" \
  --payload-format-version "2.0" \
  --query "IntegrationId" --output text)

# 6. Create catch-all route: ANY /api/{proxy+}
aws apigatewayv2 create-route \
  --api-id "$API_ID" \
  --route-key "ANY /api/{proxy+}" \
  --target "integrations/$INTEGRATION_ID"

# 7. Create default stage with auto-deploy
aws apigatewayv2 create-stage \
  --api-id "$API_ID" \
  --stage-name '$default' \
  --auto-deploy

# 8. Grant API Gateway permission to invoke Lambda
aws lambda add-permission \
  --function-name worldmonitor-api \
  --statement-id apigateway-invoke \
  --action lambda:InvokeFunction \
  --principal apigateway.amazonaws.com \
  --source-arn "arn:aws:execute-api:us-east-1:$ACCOUNT_ID:$API_ID/*/*"

API_ENDPOINT=$(aws apigatewayv2 get-api \
  --api-id "$API_ID" \
  --query "ApiEndpoint" --output text)

echo "API Gateway: $API_ENDPOINT"
```

---

### B3. Middleware → CloudFront Functions

Replace the Vercel middleware (bot filtering) with a CloudFront Function:

```bash
# 1. Create the CloudFront Function
cat > /tmp/bot-filter.js << 'FUNCEOF'
function handler(event) {
  var request = event.request;
  var uri = request.uri;
  var ua = (request.headers['user-agent'] && request.headers['user-agent'].value) || '';

  // Only filter /api/* paths
  if (!uri.startsWith('/api/')) {
    return request;
  }

  // Allow public endpoints
  if (uri === '/api/version') {
    return request;
  }

  // Block bots
  var botPattern = /bot|crawl|spider|slurp|archiver|wget|curl\/|python-requests|scrapy|httpclient|go-http|java\/|libwww|perl|ruby|php\/|ahrefsbot|semrushbot|mj12bot|dotbot|baiduspider|yandexbot|sogou|bytespider|petalbot|gptbot|claudebot|ccbot/i;

  if (botPattern.test(ua) || !ua || ua.length < 10) {
    return {
      statusCode: 403,
      statusDescription: 'Forbidden',
      headers: { 'content-type': { value: 'application/json' } },
      body: '{"error":"Forbidden"}'
    };
  }

  return request;
}
FUNCEOF

CF_FUNC_ARN=$(aws cloudfront create-function \
  --name worldmonitor-bot-filter \
  --function-config '{"Comment":"Bot filter for API routes","Runtime":"cloudfront-js-2.0"}' \
  --function-code fileb:///tmp/bot-filter.js \
  --query "FunctionSummary.FunctionMetadata.FunctionARN" --output text)

# 2. Publish the function
ETAG=$(aws cloudfront describe-function \
  --name worldmonitor-bot-filter \
  --query "ETag" --output text)

aws cloudfront publish-function \
  --name worldmonitor-bot-filter \
  --if-match "$ETAG"

# 3. Associate with CloudFront distribution
#    (Update the distribution to add the function association
#     on the API cache behavior — viewer-request event)
```

---

### B4. Cron Job → EventBridge + Lambda

```bash
# 1. Create a separate Lambda for the cron job
#    (or reuse the main Lambda with a different event path)
CRON_SECRET=$(openssl rand -hex 32)

aws lambda update-function-configuration \
  --function-name worldmonitor-api \
  --environment "Variables={
    CRON_SECRET=$CRON_SECRET,
    UPSTASH_REDIS_REST_URL=https://your-redis.upstash.io,
    UPSTASH_REDIS_REST_TOKEN=AX...your-token
  }"

# 2. Create the EventBridge rule (every 2 hours)
aws events put-rule \
  --name worldmonitor-aviation-cache-warmer \
  --schedule-expression "rate(2 hours)" \
  --state ENABLED \
  --description "Warm the aviation cache every 2 hours"

# 3. Create the target — invoke Lambda with a cron-like request
cat > /tmp/cron-input.json << 'CRONEOF'
{
  "version": "2.0",
  "routeKey": "GET /api/cron/warm-aviation-cache",
  "rawPath": "/api/cron/warm-aviation-cache",
  "headers": {
    "authorization": "Bearer CRON_SECRET_PLACEHOLDER"
  },
  "requestContext": {
    "http": { "method": "GET", "path": "/api/cron/warm-aviation-cache" }
  }
}
CRONEOF

sed -i "s/CRON_SECRET_PLACEHOLDER/$CRON_SECRET/g" /tmp/cron-input.json

aws events put-targets \
  --rule worldmonitor-aviation-cache-warmer \
  --targets "[{
    \"Id\": \"worldmonitor-api-lambda\",
    \"Arn\": \"arn:aws:lambda:us-east-1:$ACCOUNT_ID:function:worldmonitor-api\",
    \"Input\": $(cat /tmp/cron-input.json | jq -c '. | @json')
  }]"

# 4. Grant EventBridge permission to invoke Lambda
aws lambda add-permission \
  --function-name worldmonitor-api \
  --statement-id eventbridge-cron \
  --action lambda:InvokeFunction \
  --principal events.amazonaws.com \
  --source-arn "arn:aws:events:us-east-1:$ACCOUNT_ID:rule/worldmonitor-aviation-cache-warmer"
```

---

### B5. Relay Server → ECS Fargate

The relay server needs to be always-on. ECS Fargate provides managed containers
without managing EC2 instances.

```bash
# 1. Create an ECR repository
aws ecr create-repository \
  --repository-name worldmonitor-relay \
  --region us-east-1

ECR_URI=$(aws ecr describe-repositories \
  --repository-names worldmonitor-relay \
  --query "repositories[0].repositoryUri" --output text)

# 2. Create a Dockerfile for the relay
cat > /tmp/Dockerfile.relay << 'DOCKEREOF'
FROM node:20-slim

RUN apt-get update && apt-get install -y curl && rm -rf /var/lib/apt/lists/*

WORKDIR /app

COPY package.json package-lock.json ./
RUN npm ci --omit=dev

COPY scripts/ ./scripts/
COPY api/ ./api/
COPY server/ ./server/
COPY data/ ./data/

EXPOSE 3004

HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD curl -f http://localhost:3004/health || exit 1

CMD ["node", "scripts/ais-relay.cjs"]
DOCKEREOF

# 3. Build and push the Docker image
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin "$ECR_URI"

docker build -t worldmonitor-relay -f /tmp/Dockerfile.relay .
docker tag worldmonitor-relay:latest "$ECR_URI:latest"
docker push "$ECR_URI:latest"

# 4. Create ECS cluster
aws ecs create-cluster \
  --cluster-name worldmonitor-cluster \
  --capacity-providers FARGATE

# 5. Create CloudWatch log group
aws logs create-log-group \
  --log-group-name /ecs/worldmonitor-relay

# 6. Create task execution role
aws iam create-role \
  --role-name worldmonitor-ecs-execution-role \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": { "Service": "ecs-tasks.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }]
  }'

aws iam attach-role-policy \
  --role-name worldmonitor-ecs-execution-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy

ECS_ROLE_ARN=$(aws iam get-role \
  --role-name worldmonitor-ecs-execution-role \
  --query "Role.Arn" --output text)

# 7. Register task definition
RELAY_SECRET=$(openssl rand -hex 32)

cat > /tmp/task-def.json << TASKEOF
{
  "family": "worldmonitor-relay",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "512",
  "memory": "1024",
  "executionRoleArn": "$ECS_ROLE_ARN",
  "containerDefinitions": [{
    "name": "relay",
    "image": "$ECR_URI:latest",
    "portMappings": [{ "containerPort": 3004, "protocol": "tcp" }],
    "environment": [
      { "name": "PORT", "value": "3004" },
      { "name": "RELAY_SHARED_SECRET", "value": "$RELAY_SECRET" },
      { "name": "RELAY_AUTH_HEADER", "value": "x-relay-key" },
      { "name": "ALLOW_UNAUTHENTICATED_RELAY", "value": "false" },
      { "name": "UPSTASH_REDIS_REST_URL", "value": "https://your-redis.upstash.io" },
      { "name": "UPSTASH_REDIS_REST_TOKEN", "value": "AX...your-token" },
      { "name": "AISSTREAM_API_KEY", "value": "..." },
      { "name": "OPENSKY_CLIENT_ID", "value": "..." },
      { "name": "OPENSKY_CLIENT_SECRET", "value": "..." }
    ],
    "logConfiguration": {
      "logDriver": "awslogs",
      "options": {
        "awslogs-group": "/ecs/worldmonitor-relay",
        "awslogs-region": "us-east-1",
        "awslogs-stream-prefix": "relay"
      }
    },
    "healthCheck": {
      "command": ["CMD-SHELL", "curl -f http://localhost:3004/health || exit 1"],
      "interval": 30,
      "timeout": 5,
      "retries": 3,
      "startPeriod": 10
    }
  }]
}
TASKEOF

aws ecs register-task-definition \
  --cli-input-json file:///tmp/task-def.json

# 8. Create a VPC security group for the relay
VPC_ID=$(aws ec2 describe-vpcs \
  --filters "Name=isDefault,Values=true" \
  --query "Vpcs[0].VpcId" --output text)

SG_ID=$(aws ec2 create-security-group \
  --group-name worldmonitor-relay-sg \
  --description "Security group for World Monitor relay" \
  --vpc-id "$VPC_ID" \
  --query "GroupId" --output text)

aws ec2 authorize-security-group-ingress \
  --group-id "$SG_ID" \
  --protocol tcp \
  --port 3004 \
  --cidr 0.0.0.0/0

# 9. Get default subnets
SUBNETS=$(aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" "Name=default-for-az,Values=true" \
  --query "Subnets[*].SubnetId" --output text | tr '\t' ',')

# 10. Create the ECS service
aws ecs create-service \
  --cluster worldmonitor-cluster \
  --service-name worldmonitor-relay \
  --task-definition worldmonitor-relay \
  --desired-count 1 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={
    subnets=[$SUBNETS],
    securityGroups=[$SG_ID],
    assignPublicIp=ENABLED
  }"

# 11. Get the relay's public IP (after task starts)
TASK_ARN=$(aws ecs list-tasks \
  --cluster worldmonitor-cluster \
  --service-name worldmonitor-relay \
  --query "taskArns[0]" --output text)

ENI_ID=$(aws ecs describe-tasks \
  --cluster worldmonitor-cluster \
  --tasks "$TASK_ARN" \
  --query "tasks[0].attachments[0].details[?name=='networkInterfaceId'].value" \
  --output text)

RELAY_IP=$(aws ec2 describe-network-interfaces \
  --network-interface-ids "$ENI_ID" \
  --query "NetworkInterfaces[0].Association.PublicIp" --output text)

echo "Relay: http://$RELAY_IP:3004"

# 12. Update Lambda to point to the relay
aws lambda update-function-configuration \
  --function-name worldmonitor-api \
  --environment "Variables={
    WS_RELAY_URL=http://$RELAY_IP:3004,
    RELAY_SHARED_SECRET=$RELAY_SECRET,
    RELAY_AUTH_HEADER=x-relay-key
  }"
```

> **Production tip:** For a stable relay URL, place an Application Load Balancer
> (ALB) in front of the ECS service and use the ALB DNS name as `WS_RELAY_URL`.

---

### B6. Verify AWS Deployment

```bash
# Check CloudFront (frontend)
echo "Frontend: https://$DISTRIBUTION_DOMAIN"
curl -s -o /dev/null -w "%{http_code}" "https://$DISTRIBUTION_DOMAIN"
# Expect: 200

# Check API Gateway
echo "API: $API_ENDPOINT"
curl -s "$API_ENDPOINT/api/version"
# Expect: {"version":"2.5.23",...}

# Check an API endpoint with data
curl -s "$API_ENDPOINT/api/seismology/v1/list-earthquakes" | head -c 200
# Expect: JSON with earthquake data

# Check ECS relay
curl -s "http://$RELAY_IP:3004/health"
# Expect: {"ok":true,...}
```

---

## Adapter Layer — Porting Vercel Edge Functions

The existing API handlers use the **Web Standard `Request`/`Response` API**
(not Vercel-proprietary APIs), which makes porting straightforward. The main
things to adapt are:

1. **Entry point routing** — Vercel uses file-based routing (`api/foo.js` →
   `/api/foo`). Azure/AWS need explicit route registration.
2. **`export const config = { runtime: 'edge' }`** — ignored on other platforms.
3. **Client IP extraction** — Vercel injects `x-real-ip`; other platforms
   use `x-forwarded-for` or request context.

### Adapter for AWS Lambda

Create `lambda-adapter.mjs` in the project root:

```javascript
// lambda-adapter.mjs
// Adapts Vercel-style handlers (Web Request/Response) to AWS Lambda format.

import { resolve } from "path";
import { readdir } from "fs/promises";

// Dynamic handler registry — maps URL paths to handler modules.
const handlerCache = new Map();

// Build the route table on cold start.
async function buildRouteTable() {
  if (handlerCache.size > 0) return;

  // Import the gateway handlers (RPC-style)
  const rpcDirs = [
    "api/conflict/v1",
    "api/market/v1",
    "api/aviation/v1",
    "api/climate/v1",
    "api/cyber/v1",
    "api/displacement/v1",
    "api/economic/v1",
    "api/infrastructure/v1",
    "api/intelligence/v1",
    "api/maritime/v1",
    "api/military/v1",
    "api/news/v1",
    "api/prediction/v1",
    "api/research/v1",
    "api/seismology/v1",
    "api/supply_chain/v1",
    "api/wildfire/v1",
  ];

  for (const dir of rpcDirs) {
    try {
      const mod = await import(`./${dir}/[rpc].ts`);
      const prefix = `/${dir}`;
      handlerCache.set(prefix, mod.default);
    } catch (err) {
      console.warn(`[adapter] Skipping ${dir}:`, err.message);
    }
  }

  // Import standalone handlers
  const standaloneHandlers = [
    "api/bootstrap.js",
    "api/rss-proxy.js",
    "api/opensky.js",
    "api/version.js",
    "api/oref-alerts.js",
    "api/telegram-feed.js",
    "api/polymarket.js",
    "api/geo.js",
    "api/story.js",
    "api/gpsjam.js",
    "api/youtube/live.js",
    "api/youtube/embed.js",
    "api/register-interest.js",
    "api/fwdstart.js",
    "api/cron/warm-aviation-cache.ts",
  ];

  for (const file of standaloneHandlers) {
    try {
      const mod = await import(`./${file}`);
      const route = "/" + file.replace(/\.(js|ts)$/, "").replace(/\/index$/, "");
      handlerCache.set(route, mod.default);
    } catch (err) {
      console.warn(`[adapter] Skipping ${file}:`, err.message);
    }
  }
}

// Convert AWS API Gateway v2 event to Web Request
function eventToRequest(event) {
  const {
    rawPath,
    rawQueryString,
    headers,
    body,
    isBase64Encoded,
    requestContext,
  } = event;

  const url = `https://${headers.host || "localhost"}${rawPath}${
    rawQueryString ? "?" + rawQueryString : ""
  }`;

  const method = requestContext?.http?.method || "GET";

  // Inject x-real-ip from API Gateway source IP
  const sourceIp = requestContext?.http?.sourceIp;
  if (sourceIp && !headers["x-real-ip"]) {
    headers["x-real-ip"] = sourceIp;
  }

  const init = { method, headers };

  if (body && method !== "GET" && method !== "HEAD") {
    init.body = isBase64Encoded ? Buffer.from(body, "base64") : body;
  }

  return new Request(url, init);
}

// Convert Web Response to API Gateway v2 format
async function responseToResult(response) {
  const body = await response.text();
  const headers = {};
  response.headers.forEach((value, key) => {
    headers[key] = value;
  });

  return {
    statusCode: response.status,
    headers,
    body,
    isBase64Encoded: false,
  };
}

// Match a request path to a handler
function matchHandler(path) {
  // Exact match for standalone handlers
  const exact = handlerCache.get(path);
  if (exact) return exact;

  // Prefix match for RPC gateway handlers
  // e.g., /api/conflict/v1/list-acled-events → /api/conflict/v1
  for (const [prefix, handler] of handlerCache) {
    if (path.startsWith(prefix + "/") || path === prefix) {
      return handler;
    }
  }

  return null;
}

// Lambda handler entry point
export async function handler(event) {
  await buildRouteTable();

  const path = event.rawPath || event.requestContext?.http?.path || "/";
  const matchedHandler = matchHandler(path);

  if (!matchedHandler) {
    return {
      statusCode: 404,
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ error: "Not found", path }),
    };
  }

  try {
    const request = eventToRequest(event);
    const response = await matchedHandler(request);
    return await responseToResult(response);
  } catch (err) {
    console.error("[adapter] Handler error:", err);
    return {
      statusCode: 500,
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ error: "Internal server error" }),
    };
  }
}
```

### Adapter for Azure Functions (v4 model)

Create `azure-adapter.ts`:

```typescript
// azure-adapter.ts
// Adapts Vercel-style handlers to Azure Functions v4 HTTP triggers.

import {
  app,
  HttpRequest,
  HttpResponseInit,
  InvocationContext,
} from "@azure/functions";

// Import all handlers (same pattern as Lambda adapter)
import bootstrapHandler from "../../api/bootstrap.js";
import rssProxyHandler from "../../api/rss-proxy.js";
import openskyHandler from "../../api/opensky.js";
import versionHandler from "../../api/version.js";
// ... import remaining handlers

type VercelHandler = (req: Request) => Promise<Response>;

const routes: Array<{ pattern: string; handler: VercelHandler }> = [
  { pattern: "bootstrap", handler: bootstrapHandler },
  { pattern: "rss-proxy", handler: rssProxyHandler },
  { pattern: "opensky", handler: openskyHandler },
  { pattern: "version", handler: versionHandler },
  // ... register remaining routes
];

// Convert Azure HttpRequest to Web Request
function toWebRequest(azReq: HttpRequest): Request {
  const url = azReq.url;
  const method = azReq.method;
  const headers: Record<string, string> = {};

  azReq.headers.forEach((value, key) => {
    headers[key] = value;
  });

  // Inject x-real-ip from x-forwarded-for (Azure standard)
  if (!headers["x-real-ip"] && headers["x-forwarded-for"]) {
    headers["x-real-ip"] = headers["x-forwarded-for"].split(",")[0].trim();
  }

  const init: RequestInit = { method, headers };

  if (method !== "GET" && method !== "HEAD") {
    init.body = azReq.text();
  }

  return new Request(url, init);
}

// Convert Web Response to Azure response
async function toAzureResponse(response: Response): Promise<HttpResponseInit> {
  const body = await response.text();
  const headers: Record<string, string> = {};
  response.headers.forEach((value, key) => {
    headers[key] = value;
  });

  return { status: response.status, headers, body };
}

// Register a catch-all HTTP trigger
app.http("api-gateway", {
  methods: ["GET", "POST", "OPTIONS"],
  authLevel: "anonymous",
  route: "api/{*restOfPath}",
  handler: async (
    request: HttpRequest,
    context: InvocationContext
  ): Promise<HttpResponseInit> => {
    const path = request.params.restOfPath || "";

    // Match route
    const matched = routes.find((r) => path.startsWith(r.pattern));
    if (!matched) {
      return {
        status: 404,
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ error: "Not found" }),
      };
    }

    try {
      const webReq = toWebRequest(request);
      const webRes = await matched.handler(webReq);
      return await toAzureResponse(webRes);
    } catch (err) {
      context.error("Handler error:", err);
      return {
        status: 500,
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ error: "Internal server error" }),
      };
    }
  },
});
```

### Key Porting Notes

| Concern | Vercel (Current) | What to Change |
|---|---|---|
| `export const config = { runtime: 'edge' }` | Edge runtime declaration | Remove or ignore — not needed on Azure/AWS |
| `request.headers.get('x-real-ip')` | Vercel injects trusted IP | Populate from `x-forwarded-for` (Azure/AWS) or `requestContext.http.sourceIp` (AWS) |
| `process.env.VERCEL_ENV` | Deploy environment detection | Replace with `process.env.NODE_ENV` or your own env var |
| `Response.json(...)` | Web API shorthand | Available in Node.js 20+; works as-is |
| `new Response(body, {status, headers})` | Web API | Available in Node.js 20+; works as-is |
| `new URL(request.url)` | URL parsing | Works as-is in Node.js 20+ |
| `AbortSignal.timeout(ms)` | Request timeouts | Available in Node.js 20+; works as-is |
| `crypto.subtle` | Timing-safe comparison | Available in Node.js 20+; works as-is |
| Upstash Redis REST | Cache via HTTP fetch | Works from any cloud — no changes needed |
| `@upstash/ratelimit` | Rate limiting library | Works from any cloud — no changes needed |

> **Good news:** Because the codebase uses **Web Standard APIs** (not Vercel
> SDK imports), the actual handler logic requires **zero changes**. Only the
> entry point routing and request/response wrapping need adaptation.

---

## DNS & Custom Domain Setup

### Azure Custom Domain

```bash
# Add custom domain to Static Web App
az staticwebapp hostname set \
  --name swa-worldmonitor \
  --resource-group rg-worldmonitor \
  --hostname www.yourdomain.com

# Or use Azure Front Door for apex domain
az afd custom-domain create \
  --custom-domain-name yourdomain \
  --profile-name afd-worldmonitor \
  --resource-group rg-worldmonitor \
  --host-name yourdomain.com \
  --certificate-type ManagedCertificate
```

DNS records:
```
Type   Name   Value
CNAME  www    swa-worldmonitor.azurestaticapps.net
       OR     worldmonitor.z01.azurefd.net (if using Front Door)
TXT    _dnsauth.www   <validation token from Azure>
```

### AWS Custom Domain

```bash
# 1. Request ACM certificate (must be in us-east-1 for CloudFront)
CERT_ARN=$(aws acm request-certificate \
  --domain-name yourdomain.com \
  --subject-alternative-names "*.yourdomain.com" \
  --validation-method DNS \
  --region us-east-1 \
  --query "CertificateArn" --output text)

# 2. Get DNS validation records
aws acm describe-certificate \
  --certificate-arn "$CERT_ARN" \
  --region us-east-1 \
  --query "Certificate.DomainValidationOptions[*].ResourceRecord"

# 3. After DNS validation, add to CloudFront distribution
aws cloudfront update-distribution \
  --id "$DISTRIBUTION_ID" \
  --distribution-config <updated-config-with-aliases-and-cert>
```

DNS records:
```
Type    Name              Value
A       yourdomain.com    <CloudFront distribution>.cloudfront.net (ALIAS)
CNAME   www               <CloudFront distribution>.cloudfront.net
CNAME   api               <API Gateway ID>.execute-api.us-east-1.amazonaws.com
```

---

## Monitoring & Observability

### Azure

```bash
# Create Application Insights
az monitor app-insights component create \
  --app ai-worldmonitor \
  --location eastus2 \
  --resource-group rg-worldmonitor

# Link to Function App
INSTRUMENTATION_KEY=$(az monitor app-insights component show \
  --app ai-worldmonitor \
  --resource-group rg-worldmonitor \
  --query "instrumentationKey" -o tsv)

az functionapp config appsettings set \
  --name func-worldmonitor \
  --resource-group rg-worldmonitor \
  --settings APPINSIGHTS_INSTRUMENTATIONKEY="$INSTRUMENTATION_KEY"

# View live logs
az webapp log tail \
  --name app-worldmonitor-relay \
  --resource-group rg-worldmonitor
```

### AWS

```bash
# Lambda logs
aws logs tail /aws/lambda/worldmonitor-api --follow

# ECS relay logs
aws logs tail /ecs/worldmonitor-relay --follow

# Create CloudWatch alarm for Lambda errors
aws cloudwatch put-metric-alarm \
  --alarm-name worldmonitor-api-errors \
  --namespace AWS/Lambda \
  --metric-name Errors \
  --dimensions Name=FunctionName,Value=worldmonitor-api \
  --statistic Sum \
  --period 300 \
  --threshold 10 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 1 \
  --alarm-actions "arn:aws:sns:us-east-1:$ACCOUNT_ID:worldmonitor-alerts"
```

---

## Cost Estimates

### Azure (Monthly)

| Component | SKU | Estimated Cost |
|---|---|---|
| Static Web Apps | Free tier | $0 |
| Azure Functions | Consumption (1M exec/mo free) | $0–20 |
| App Service (relay) | B1 (1 core, 1.75 GB) | ~$13 |
| Front Door | Standard | ~$35 |
| **Total** | | **~$48–68/mo** |

### AWS (Monthly)

| Component | SKU | Estimated Cost |
|---|---|---|
| S3 | Standard (< 1 GB) | < $1 |
| CloudFront | Free tier (1 TB/mo) | $0–10 |
| Lambda | 1M requests free, then $0.20/M | $0–5 |
| API Gateway | 1M requests free, then $1/M | $0–5 |
| ECS Fargate (relay) | 0.5 vCPU / 1 GB | ~$18 |
| EventBridge | Free tier (14M events/mo) | $0 |
| **Total** | | **~$19–39/mo** |

> **Note:** Upstash Redis is billed separately (~$0 on free tier, ~$10/mo
> for Pro). These estimates assume moderate traffic (~100K API calls/month).

---

## Troubleshooting

### Common Issues

**1. API returns 403 Forbidden**
- Check CORS configuration. The handlers validate `Origin` headers against an
  allowlist in `api/_cors.js`. You need to add your new domain to
  `ALLOWED_ORIGIN_PATTERNS`.

```javascript
// api/_cors.js — add your domain
const ALLOWED_ORIGIN_PATTERNS = [
  /^https:\/\/(.*\.)?worldmonitor\.app$/,
  /^https:\/\/(.*\.)?yourdomain\.com$/,  // ← add this
  /^https?:\/\/localhost(:\d+)?$/,
];
```

**2. Redis cache misses on new deployment**
- The codebase prefixes Redis keys with `VERCEL_ENV` and `VERCEL_GIT_COMMIT_SHA`
  for preview deploys. On Azure/AWS, these env vars won't exist, so keys will
  default to unprefixed (production behavior). This is correct — no action needed.

**3. Relay server connection refused**
- Ensure `WS_RELAY_URL` is set on the API function app and points to the
  relay's public URL.
- Ensure `RELAY_SHARED_SECRET` matches on both the API functions and the relay.
- Ensure the relay's port (3004) is open in security groups / firewall rules.

**4. Cron job not firing**
- Azure: Check Timer Trigger logs in Application Insights.
- AWS: Check EventBridge rule status: `aws events describe-rule --name worldmonitor-aviation-cache-warmer`
- Verify `CRON_SECRET` env var is set and matches the `Authorization` header.

**5. Static site returns 404 on refresh**
- SPA routing requires all paths to fall back to `index.html`.
- Azure: Ensure `navigationFallback` is configured in `staticwebapp.config.json`.
- AWS: Ensure CloudFront custom error response maps 404 → `/index.html` with 200 status.

**6. YouTube/RSS feeds blocked**
- Some upstream feeds block cloud provider IP ranges. The relay server
  (`scripts/ais-relay.cjs`) supports residential proxy configuration via
  environment variables for YouTube and OREF endpoints.

**7. ONNX/Transformers.js WASM errors**
- The frontend loads WASM files for client-side ML inference. Ensure your CDN
  serves `.wasm` files with `Content-Type: application/wasm`. S3 does this
  by default; Azure Static Web Apps may need a MIME type override.

---

## Summary Checklist

Use this checklist to verify your deployment is complete:

- [ ] Frontend built (`npm run build`) and deployed to CDN
- [ ] SPA fallback routing configured (404 → index.html)
- [ ] Cache headers set (assets: immutable, index.html: no-cache)
- [ ] API functions deployed with adapter layer
- [ ] All environment variables set on API function app
- [ ] CORS allowlist updated with your domain
- [ ] Relay server deployed and reachable
- [ ] `WS_RELAY_URL` + `RELAY_SHARED_SECRET` synchronized
- [ ] Cron job configured (every 2 hours)
- [ ] `CRON_SECRET` set and matching
- [ ] Custom domain + SSL configured
- [ ] Health checks passing (`/api/version`, relay `/health`)
- [ ] Monitoring/alerting configured
