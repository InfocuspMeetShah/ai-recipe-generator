# Learning notes: AI Recipe Generator on AWS

Personal notes while following the AWS hands-on tutorial
[Build a serverless web application using generative AI](https://docs.aws.amazon.com/hands-on/latest/build-serverless-web-app-lambda-amplify-bedrock-cognito-gen-ai/build-serverless-web-app-lambda-amplify-bedrock-cognito-gen-ai.html).

The goal is to understand the concepts and terminology, not just to get the app running.

## What we're building

Type a few ingredients, and an AI model suggests a recipe.

| Service | Role in the app | In plain words |
|---|---|---|
| **Amplify** | Framework and hosting that ties everything together | Like Vercel or Netlify, plus a backend builder |
| **Cognito** | Sign-up and login | A user database you don't have to build |
| **AppSync** | GraphQL API | The front door the app calls to ask for a recipe |
| **Resolver / Lambda** | Code that runs when the API is called | Serverless code: no servers to manage |
| **Bedrock** | Runs the AI model (Claude) | AWS's service for using AI models |

## Glossary

- **Account:** a separate container for AWS resources and billing. Separate accounts can't affect each other.
- **Region:** the physical data-center location where resources run (for example `us-east-1`, N. Virginia). Most services are separate in each region.
- **ARN (Amazon Resource Name):** the unique ID of any AWS resource, in the form `arn:aws:<service>:<region>:<account>:<resource>`.
- **IAM (Identity and Access Management):** decides who can do what in AWS.
- **IAM role:** a set of permissions you or a service "put on" temporarily. Unlike a permanent user, it has no long-lived password or keys.
- **SSO / IAM Identity Center:** signs you in once through a browser and hands out temporary role credentials (that's what `aws sso login` does).
- **AWS CLI profile:** a named set of settings in `~/.aws/config` that tells the CLI which account and role to use (the `AWS_PROFILE` variable picks one).
- **Serverless:** you never create or patch machines. AWS runs your code, scales it, and charges only for usage.

## Local setup

- Node.js and npm, installed with nvm.
- AWS CLI with an SSO profile, chosen through `AWS_PROFILE` and `AWS_DEFAULT_REGION` in `~/.bashrc`.
- Check who you're signed in as: `aws sts get-caller-identity`.
- Sign in again when the temporary credentials expire: `aws sso login`.

## Step log

### Step 1: Create the React app (local only)

```bash
npm create vite@latest ai-recipe-generator -- --template react-ts -y
cd ai-recipe-generator
npm install
npm run dev
```

- **Vite** creates a starter project and runs a fast local dev server.
- **React + TypeScript** (`react-ts`): the UI library, plus JavaScript with types.
- `npm install` downloads the libraries listed in `package.json` into `node_modules/`.
- `npm run dev` serves the app at `http://localhost:5173` for local testing.
- No AWS involved yet. This is just the frontend.

### Step 2: Push the code to GitHub

- **Why:** Amplify Hosting deploys from a Git repository, not from uploaded files.
- **CI/CD:** each push is built automatically (Continuous Integration) and goes live if the build succeeds (Continuous Deployment).
- `git push --force` replaces what's on GitHub with your local version. It was needed once because the old repo had a placeholder commit. Use it with care, because it can erase other people's work.

### Bedrock model access (differs from the tutorial)

The tutorial is out of date here:

- **No "Request model access" button anymore.** Bedrock models are enabled automatically the first time they're called. Anthropic models also need a one-time **use case form** per account (already submitted for this account).
- **Claude 3.5 Sonnet is retired.** We use **Claude Sonnet 4.6** instead.
- **Inference profile:** current Claude models must be called through an inference profile, not the plain model ID.
  - A **cross-region inference profile** such as `us.anthropic.claude-sonnet-4-6` sends each request to us-east-1, us-east-2 or us-west-2, whichever has capacity.
  - There's also a `global.` profile that can use any region worldwide.
- **Effect on the code (Step 3):**
  - Model ID in `bedrock.js`: `us.anthropic.claude-sonnet-4-6`.
  - The IAM policy must allow the inference profile **and** the foundation model in every region it can route to.
- **Playground:** try a model in the console before writing any code.
- Useful CLI checks:
  - `aws bedrock list-foundation-models --by-provider anthropic`
  - `aws bedrock get-foundation-model-availability --model-id anthropic.claude-sonnet-4-6`

### Step 1b: Amplify Gen 2 scaffold and hosting

- `npm create amplify@latest -y` creates the `amplify/` folder: the backend defined as TypeScript code.
- **Infrastructure as Code (IaC):** describe resources in code instead of clicking through the console. The setup can be repeated, reviewed in Git, and each developer can get their own copy. Amplify uses **AWS CDK** and **CloudFormation** underneath.
- **CloudFormation stack:** one bundle of resources that is created, updated and deleted together.

| File | Defines | Becomes in AWS |
|---|---|---|
| `amplify/auth/resource.ts` | Login with email and a verification code | **Cognito User Pool** |
| `amplify/data/resource.ts` | API schema and access rules | **AppSync GraphQL API** (plus a **DynamoDB** table for each `model`) |
| `amplify/backend.ts` | Entry point that combines everything | The CloudFormation stack |

- `allow.guest()` lets anyone, even signed-out users, use the API. `allow.authenticated()` allows signed-in users only.
- **Amplify Hosting:** the app watches the GitHub `main` branch and builds, deploys and serves the site from a global **CDN** on every push.
- **Service role:** an IAM role that an *AWS service* (here, Amplify) assumes, so it can create resources for you.
- **Lesson learned:** the app was first created in eu-north-1 by mistake, because the console only shows the region you've selected. Before creating anything, check the **account and region in the top-right corner**.
- **New-account verification:** brand-new accounts may get `AccessDeniedException` ("account is being verified") on Bedrock for up to about 2 hours. This is unrelated to IAM.

### Step 3: Connect the backend to Bedrock

**`amplify/data/bedrock.js`: the resolver**

- A **resolver** is the code that runs when an API operation is called.
  - `request()` turns the GraphQL call into an HTTP request to Bedrock.
  - `response()` turns Bedrock's reply into the GraphQL result.
- This is an **AppSync JavaScript resolver**. It runs inside AppSync, with no Lambda function needed.
- Changed from the tutorial: `resourcePath: /model/us.anthropic.claude-sonnet-4-6/invoke`.
- The request body follows Anthropic's **Messages API** format (`anthropic_version`, `max_tokens`, `messages`).

**`amplify/backend.ts`: the data source and the permission**

- **HTTP data source:** tells AppSync where resolvers send requests (the Bedrock runtime endpoint in us-east-1).
- **SigV4 signing** (`authorizationConfig`): AppSync signs each request with AWS credentials so Bedrock knows who's calling.
- **IAM policy statement:** **Effect** (allow), **Action** (`bedrock:InvokeModel`), **Resource** (which ARNs).
- **Least privilege:** grant only the exact action on the exact resources needed.
- Calling through an inference profile needs permission on:
  - the **inference profile ARN**, which includes your account ID (filled in by `backend.data.stack.account`), and
  - the **foundation model ARN in every region the profile can route to** (us-east-1, us-east-2 and us-west-2). These ARNs have no account ID, because AWS owns the models.
- If one region's ARN is missing, some requests randomly fail with an IAM `AccessDeniedException`.

### Step 4: Define the API query and deploy the sandbox

**`amplify/data/resource.ts`**

- `a.customType` describes the shape of a response and creates **no database table**. An `a.model` creates a DynamoDB table.
- `a.query()` is a read-style GraphQL operation. A *mutation* is one that changes data.
- `.handler(a.handler.custom({ entry, dataSource }))` connects the query to the resolver (`bedrock.js`) and the data source (`bedrockDS`).
- **Authorization modes:**
  - The API's default mode is an **API key** (a shared secret that expires after 30 days).
  - `askBedrock` overrides it with `allow.authenticated()`, so it needs a **Cognito user pool** login.
  - The deployed API supports `AMAZON_COGNITO_USER_POOLS` and `AWS_IAM` besides the API key.

**`npx ampx sandbox`**

- Creates a **personal, temporary backend** for development, separate from production (the `main` branch on Amplify Hosting).
- **Watches** the `amplify/` folder and redeploys whenever you save a file.
- **CDK bootstrap:** a one-time setup for each account and region that creates the `CDKToolkit` stack (an S3 staging bucket plus deploy roles). After bootstrapping, the sandbox stops, so run it again.
- **Synthesize:** CDK turns the TypeScript into a CloudFormation template, then deploys the stack.
  - Here, that's `amplify-airecipegenerator-meetshah-sandbox-...`, with **nested stacks** for auth and data.
- **`amplify_outputs.json`:** tells the frontend where the backend is (region, user pool ID, GraphQL URL).
  - Until the first deploy finishes, it's only a `{"version": ...}` placeholder.
  - It's **git-ignored**, because each environment (sandbox or a hosted branch) generates its own copy. Amplify Hosting creates it during the build.

### Step 5: Frontend, and debugging the first run

- `aws-amplify` is the client library that reads `amplify_outputs.json`. `@aws-amplify/ui-react` provides ready-made UI components, such as `<Authenticator>`, a complete sign-up and sign-in flow.
- `generateClient<Schema>({ authMode: "userPool" })` creates a typed API client that sends the logged-in user's Cognito token.
- **White screen:** `import { Schema }` (a type) without the `type` keyword was kept in the code sent to the browser, and the browser crashed with a `SyntaxError`. Use `import type { ... }`. For a white screen, always check **F12 → Console** first.
- **Generate kept loading:** the app only `console.log`s API errors, so the page showed nothing.
  - The real error was a Bedrock `AccessDeniedException` about **AWS Marketplace** permissions.
  - Third-party models (such as Anthropic's) need a one-time **Marketplace subscription for each model in each account**. It's triggered by the first call, and the caller needs `aws-marketplace:Subscribe`.
  - Sonnet 4.5 was already subscribed in this account and Sonnet 4.6 wasn't, so we switched to `us.anthropic.claude-sonnet-4-5-20250929-v1:0`. (4.5 is marked LEGACY and will be retired eventually.)
- **Changing the model means changing two places together:** the `resourcePath` in `bedrock.js` (what you call) and the ARNs in `backend.ts` (what you're allowed to call). If they don't match, you get an IAM `AccessDeniedException`.
- **Isolating a problem:** call the model directly from the CLI with `aws bedrock-runtime invoke-model` to tell an app bug apart from an access problem.
