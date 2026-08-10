---
sidebar_position: 18
---

# Firebase Integration

KMPStarterKit has a support for **Firebase** as a serverless backend for AI integration and other features like **Authentication**, **Push Notifications**, and **Landing Page Hosting**. Follow these steps carefully to set up Firebase and connect it with your app.


## 1. Create New or Choose Existing a Firebase Project

1. Open the Firebase console: https://console.firebase.google.com/  
2. Click **Add Project** (or select an existing one).  
3. Give your project a **unique project ID** and a **project name**.  
4. Click **Continue** and follow the prompts.  

> ⚠️ Important: For backend functions, you will need **Blaze (pay-as-you-go) plan**. Firebase has generous free limits and it’s cheap for small projects. In order to enable **Firebase Blaze Plan** Go to **Project Overview → Usage and billing → Details&Settings →  Modify Plan → Blaze plan**. This is required for Firebase Functions (serverless backend) to work correctly.




## 2. Store your API Keys Securely

To securely store and access your API keys, [**Google Cloud Secret Manager**](https://console.cloud.google.com/security/secret-manager) is used. This is a secure and scalable way to manage sensitive information like API keys. Below are the steps to retrieve and store your keys securely.

### 2.1 Enable the Secret Manager API

Cloud Functions can't read secrets until this API is enabled. Open the API library page and click **Enable** (replace `YOUR_PROJECT_ID`):

`https://console.cloud.google.com/apis/library/secretmanager.googleapis.com?project=YOUR_PROJECT_ID`

Then open the [**Secret Manager**](https://console.cloud.google.com/security/secret-manager) console. Make sure you chose the correct project.


### 2.2 Get your API Keys

1. **OpenAI API Key**:
   - To use OpenAI’s API, you need an API key. You can obtain it by signing up for OpenAI at [OpenAI's API page](https://platform.openai.com/signup).
   - After signing in, go to the **API Keys** section under your account settings to create and retrieve your API key.

2. **Replicate API Key**:
   - To use Replicate's API, create an account on [Replicate](https://replicate.com/).
   - Once logged in, go to the **API** section of your account dashboard, where you can generate and retrieve your API key.

### 2.3 Store API Keys

Once you have your API keys, you can securely store them in [Google Cloud Secret Manager](https://console.cloud.google.com/security/secret-manager).

1. **Navigate to Secret Manager**: 
 - https://console.cloud.google.com/security/secret-manager?project=YOUR_PROJECT_ID. Replace `YOUR_PROJECT_ID` with your project id.
   
2. **Create Secrets** (before deploying):
   - Click **Create Secret** and enter the exact name. Create `REPLICATE_API_KEY` (the default provider); add `OPENAI_API_KEY` only if you enable OpenAI (see [4.5 Choosing which providers to deploy](#45-choosing-which-providers-to-deploy)).
   - Paste your API key in the **Secret Value** field and click **Create**. Creating secrets before `firebase deploy` avoids interactive prompts and 403 errors.

3. **Grant Access**:
   - Ensure that the service account running your application has **access to the secret**. You can grant access by setting the appropriate permissions to allow your app to retrieve secrets.



## 2. Install Firebase CLI in your device

Open your terminal and install Firebase CLI globally in your device. Make sure you have **Node.js** installed — the functions run on the **Node 22** runtime, so use an active LTS (Node 20 or 22) to avoid old-runtime and modern-engine deploy errors. 

```bash
npm install -g firebase-tools
```

Or, with no Node required (macOS/Linux): `curl -sL https://firebase.tools | bash`.

## 3. Login to Firebase in Your Project
Navigate to your Project's `Web` root folder and login to Firebase:

```bash
firebase login
```
Follow the browser prompts to authorize Firebase CLI access.

## 4. Configure Necessary Features

Before starting, you need to initialize Firebase in your project. Make sure you followed above steps and you are in your Project's **Web** directory.


### 4.1 Backend - Firebase Functions Setup

The AI backend functions are pre-configured to support integration with popular APIs like **[Replicate](https://replicate.com/)** and **[OpenAI](https://platform.openai.com/docs/api-reference/)** securely, implemented in the `functions/index.js` file. To keep the first deploy zero-config, **only Replicate is enabled by default** — you need just one key (`REPLICATE_API_KEY`). Choose which providers deploy with the `AI_PROVIDERS` environment variable, no code editing (see [4.5 Choosing which providers to deploy](#45-choosing-which-providers-to-deploy)).

#### 1. Initialize Firebase Functions

Set up Firebase Functions:
```bash
firebase init functions
```
1. Choose your Firebase project or create a new one.
2. When prompted:
   - Select `JavaScript` as the language for functions.
   - Install dependencies when asked.
   - When asked, to override some files, select **NO**.


#### 2. Deployment
Deploy your backend functions to Firebase:

```bash
  firebase deploy --only functions
```

> After deploy is finished you will see your `CLOUD_FUNCTIONS_URL` in terminal. You will need this URL.  `CLOUD_FUNCTIONS_URL` should be something like: `https://REGION-PROJECT_ID.cloudfunctions.net`. By default `REGION` value is `us-central1`. Make sure, in your mobile app `CLOUD_FUNCTIONS_URL` inside `root/AppConfiguration.kt` file is same as what you see.

:::note First deploy on a brand-new project fails?
Two common one-time causes: the **Secret Manager API isn't enabled** (step 2.1), or the project has **no default resource location** yet — the deploy uploads its source bundle to a Cloud Storage staging bucket that a fresh project doesn't have, so it fails with `Failed to make request to generateUploadUrl`. Open **Storage → Get Started** in the Firebase Console and pick a region to set the default location (you don't use Cloud Storage in the app — this is only to let Functions deploy). Fix both, then re-run the deploy.
:::

#### 3. Test Locally
You can test your app backend locally before deploying it:
```bash
firebase emulators:start --only functions
```

#### 4. Available AI Functions

1. **Replicate API Functions**:
   These functions handle interactions with the Replicate API.
   - `replicateCreatePrediction`
   - `replicateCreateModelPrediction`
   - `replicateGetPredictionStatus`
   - `replicateCancelPrediction`

   You can find these functions in the `api/replicate.js` file. Replicate API documentation for request and response: https://replicate.com/docs/topics/predictions 

2. **OpenAI API Functions**:
   These functions interact with the OpenAI API for generating text and images.
   - `openAiCreateTextCompletion`
   - `openAiCreateImage`

   These are located in the `api/openai.js` file. OpenAI API documentation for request and response: https://platform.openai.com/docs/api-reference/chat


#### 5. Choosing which providers to deploy

You don't need to edit code to enable/disable a provider. `functions/index.js` reads the `AI_PROVIDERS` environment variable (comma-separated) and only exports the listed providers' functions. An unexported function doesn't bind its secret, so **you only need the API key for the provider(s) you enable**.

| `AI_PROVIDERS` | Deployed | Secret required |
|---|---|---|
| *(unset — the default)* | Replicate only | `REPLICATE_API_KEY` |
| `openai` | OpenAI only | `OPENAI_API_KEY` |
| `replicate,openai` | Both | both |

To change the default, copy `functions/.env.example` to `functions/.env` and set the variable:

```bash
# functions/.env
AI_PROVIDERS=replicate,openai
```

API keys are **not** set in `.env` — they stay in Secret Manager. Local `.env` files are gitignored.

#### 6. API Response Structure

All API responses are returned in JSON format. The structure follows this format:

```json
{
  "statusCode": <statusCode>,
  "errorMessage": <errorMessage>,
  "data": <data>
}
```

- `statusCode`: Represents the HTTP status code of the response.
- `errorMessage`: If an error occurs, this field contains the error message. If there’s no error, this field will be empty or `null`.
- `data`: This field contains the actual response data returned by the AI API, such as predictions, results, or generated content based on each AI api's response object.

#### 7. Validation

By default, all API requests require **user authentication** via Firebase, primarily to secure access to the AI APIs due to their potential cost. Before making any API request, the user must be authenticated. If you want to allow unauthenticated access to any endpoint (e.g., for testing or development purposes), you can disable authentication by setting `requireAuth: false` in the validation function:

```javascript
createPrediction: onRequest(async (req, res) => {
    cors(req, res, async () => {
        if (!await Validation.validateAll(req, res, { requireAuth: false })) return; <--- IN THIS LINE WE SET AUTH REQUIREMENT FALSE
        await makeApiRequest(`${REPLICATE_API_BASE_URL}/predictions`, "post", getReplicateApiKey(), req.body, res);
    });
}),
```

### 4.2 Authentication - Firebase Authentication Setup
1. Go to **Authentication** → **Sign-in methods** in Firebase Console.
2. Enable **Anonymous Auth** for easy access.
3. Optionally, enable **Google** and **Apple Sign-In** if your app requires them.
4. Make sure your app correctly handles authenticated users when calling Firebase Functions.

For more detailed information you can check  **[Auth](auth)** section.

### 4.3 Notifications - Firebase Push Notifications Setup
1. Go to **Messaging** in Firebase Console.
2. Test push notifications locally or on a real device before production.

For more detailed information you can check  **[Notifications](notifications)** section.

### 4.4 App Landing Page - Firebase Hosting Setup

This is a simple app landing page template that you can use to showcase your app and deploy it into Firebase easily and free. It has sections for a Hero, Problem, Features, and a Call-to-Action (CTA). It includes template Privacy Policy and Terms and Service files as well (but you should write your own according to your app, as this is just a template and not a legal document).

<YouTube id="umuaJG4AO_Q" title="App Landing Page setup" />

#### 1. Initialize Firebase Hosting

Set up Firebase Hosting:
```bash
firebase init hosting
```
1. Choose your Firebase project or create a new one.
2. When prompted:
   - **What do you want to use as your public directory?** Press **Enter** (default: `public`).
   - **Configure as a single-page app?** Press **N** (No).
   - **Set up automatic builds and deploys with GitHub?** Press **N** (No).
   - **File public/index.html already exists. Overwrite?** Press **N** (No).


#### 2. Customize

Update `public/images/hero.png`, `public/images/logo.png`, `public/images/favicon.ico` with your own app images.  
Edit the `public/config.js` file to customize the app's details. Check out **[App Landing Page](app-landing-page)** section for more details.


#### 3. Deployment
Deploy your landing page to Firebase:
  ```bash
  firebase deploy --only hosting
  ```


#### 4. Test Locally
You can test your landing page locally before deploying it:
```bash
firebase serve
```


