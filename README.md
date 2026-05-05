# UMD IFAD Portal

A React-based web portal for the UMD International Field Application Development (IFAD) program. The portal manages the full lifecycle of the student-host matching system — from host registration and student applications through admin-controlled matching, communications, and timeline management.

## Overview

The system supports three user roles:

- **Students** — browse hosts, submit applications, track status
- **Hosts** — register as experience providers, manage profiles and semester availability
- **Admins** — manage applications, run matching, control program timeline, send communications

Students authenticate exclusively via UMD CAS (Single Sign-On). Hosts and admins use AWS Cognito with email/password.

## Tech Stack

| Layer | Technology |
|---|---|
| Frontend | React 18, TypeScript, Vite |
| Routing | React Router v6 |
| Styling | Tailwind CSS (UMD brand colors) |
| Auth (students) | UMD CAS via custom JWT flow |
| Auth (hosts/admins) | AWS Cognito via AWS Amplify |
| Backend | AWS Lambda (Node.js/TypeScript), API Gateway |
| Database | DynamoDB |
| Infrastructure | AWS CDK v2 |
| Email | AWS SES |
| Storage | AWS S3 + CloudFront |
| AI features | AWS Bedrock |

## Project Structure

```
├── src/
│   ├── components/         # Shared UI and layout components
│   ├── hooks/              # useAuth and other custom hooks
│   ├── pages/
│   │   ├── admin/          # Admin dashboard pages
│   │   ├── host/           # Host-facing pages
│   │   └── student/        # Student-facing pages
│   ├── services/           # API calls, CAS auth, AWS timeline
│   └── types/              # TypeScript interfaces
├── backend/
│   ├── bin/                # CDK app entrypoint
│   ├── lib/                # CDK stack definition (ifad-stack.ts)
│   ├── lambda/functions/   # Lambda handlers (auth, admin, applications, users, public)
│   └── test/               # Jest tests
├── tests/                  # Manual API smoke test scripts
└── public/                 # Static assets
```

## Prerequisites

- Node.js 18+
- AWS CLI configured with appropriate credentials
- AWS CDK v2 installed globally (`npm install -g aws-cdk`)

## Frontend Setup

```bash
npm install
```

Create a `.env.local` file:

```env
VITE_API_URL=https://<your-api-gateway-id>.execute-api.us-east-1.amazonaws.com/prod
```

`.env.staging` and `.env.production` follow the same format and are selected via Vite's `--mode` flag.

### Development server

```bash
npm run dev
```

> **Important:** The dev server binds to `localtest.dev.umd.edu:5173` (see [vite.config.ts](vite.config.ts)). This hostname resolves to `127.0.0.1` and must be registered with UMD IT as an allowed CAS service URL before CAS login will work locally. Non-CAS flows (host/admin login) work on plain `localhost`.

### Build

```bash
npm run build              # production build
npm run build:staging      # staging build
npm run build:production   # production build (explicit)
```

### Lint

```bash
npm run lint
```

## Backend Setup

```bash
cd backend
npm install
```

Copy the example env file and fill in your AWS credentials:

```bash
cp .env.example .env
```

```env
AWS_ACCESS_KEY_ID=...
AWS_SECRET_ACCESS_KEY=...
AWS_REGION=us-east-1
STAGE=staging
```

`USER_POOL_ID`, `USER_POOL_CLIENT_ID`, `TABLE_NAME`, and `API_URL` are populated automatically after the first CDK deployment.

### Deploy backend

```bash
# Staging
npm run deploy:staging       # Linux/Mac
npm run deploy:staging:win   # Windows

# Production
npm run deploy:prod          # Linux/Mac
npm run deploy:prod:win      # Windows
```

CDK deploys the full stack: DynamoDB table, Cognito User Pool, Lambda functions, API Gateway, S3 buckets, CloudFront distribution, and SES configuration.

## Frontend Deployment

After building, sync `dist/` to S3 and invalidate CloudFront:

```bash
npm run deploy:frontend:staging
npm run deploy:frontend:prod
```

These commands require AWS CLI credentials in your shell environment.

## CAS Authentication

Students log in via UMD's Central Authentication Service (CAS). The flow:

1. User clicks "Login with UMD CAS" → browser redirects to `https://shib.idm.umd.edu/.../cas/login?service=<your-app-url>`
2. UMD authenticates the user and redirects back with `?ticket=ST-xxxxx`
3. The frontend detects the ticket and calls `POST /auth/cas/validate`
4. The Lambda validates the ticket against UMD's CAS server, parses the XML response, and returns a signed JWT (6-hour expiry)
5. The JWT is stored in `localStorage` and used for all subsequent API calls

### Setting up CAS for a new deployment

1. Register your domain(s) with UMD IT as allowed CAS service URLs — both production and any staging/dev URLs
2. For local development, register `http://localtest.dev.umd.edu:5173` with UMD IT
3. Ensure `CAS_BASE_URL` is set to `https://shib.idm.umd.edu/shibboleth-idp/profile/cas` on the Lambda (CDK sets this automatically)
4. Set a strong `JWT_SECRET` via CloudFormation parameter — this is prompted during CDK deployment

## Environment Variables

### Frontend

| Variable | Description |
|---|---|
| `VITE_API_URL` | Base URL of the deployed API Gateway |

### Backend (Lambda environment, managed by CDK)

| Variable | Description |
|---|---|
| `CAS_BASE_URL` | UMD CAS server base URL |
| `JWT_SECRET` | Secret for signing JWTs issued after CAS login |
| `TABLE_NAME` | DynamoDB table name |
| `USER_POOL_ID` | Cognito User Pool ID |
| `USER_POOL_CLIENT_ID` | Cognito App Client ID |
| `AWS_REGION` | AWS region |

## Testing

The backend has Jest configured:

```bash
cd backend
npm test
```

Tests live in `backend/test/`. The suite is currently a placeholder — coverage has not yet been written.

Manual API smoke tests are in `tests/` and exercise real API endpoints against a deployed environment:

```bash
# Requires VITE_API_URL set in a .env file at repo root
node tests/api-workflow.js
node tests/api-login-semester.js
```

## Key Files

| File | Purpose |
|---|---|
| [src/App.tsx](src/App.tsx) | Route definitions, `CASCallbackHandler`, `ProtectedRoute` |
| [src/hooks/useAuth.ts](src/hooks/useAuth.ts) | Auth state — CAS + Cognito, token management |
| [src/services/casAuth.ts](src/services/casAuth.ts) | CAS login redirect, ticket validation, logout |
| [src/services/api.ts](src/services/api.ts) | Typed API client for all backend calls |
| [backend/lib/ifad-stack.ts](backend/lib/ifad-stack.ts) | Full AWS CDK stack definition |
| [backend/lambda/functions/auth/index.ts](backend/lambda/functions/auth/index.ts) | Auth Lambda — CAS validation, Cognito flows, JWT issuance |
