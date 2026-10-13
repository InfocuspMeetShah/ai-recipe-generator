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
