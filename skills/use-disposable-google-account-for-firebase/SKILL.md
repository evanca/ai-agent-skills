---
name: use-disposable-google-account-for-firebase
description: Use when given a temporary, disposable or lab Google account (hackathon, workshop, codelab or sponsored credits) to test Firebase, or when standing up Firebase AI Logic with App Check on a short-lived Google Cloud project.
---

# Use a Disposable Google Account for Firebase

Get a hello world that calls Gemini through Firebase AI Logic, with App Check
verifying every request, inside a lab project that will be deleted. Keep the
owner's permanent projects, billing, sign-ins and committed configuration
untouched, and leave evidence that requests reached the lab project.

## Ground rules

- **The human signs in.** Never type the lab password or a one-time code. Keep
  credentials out of files, issues, logs, screenshots and other agents' prompts.
- **Record the expiry first.** Lab access often counts from issuance, not first
  login. Everything stored only in the project is deleted with it, so keep code,
  prompts and results in the repository.
- **Use the assigned project.** Do not create projects, attach personal billing
  or move a permanent project into the lab account.
- **Isolate the session.** Use a separate browser profile or an incognito window
  for the lab account. For the CLI, use a separate gcloud configuration so the
  owner's default account and project stay active:

  ```sh
  gcloud config configurations create lab --no-activate
  CLOUDSDK_ACTIVE_CONFIG_NAME=lab gcloud auth login --no-launch-browser
  ```

  Do not add `--update-adc`; it replaces the owner's application default
  credentials. The Firebase CLI is not needed for anything below.
- **Point the app at the lab through ignored config.** Use an untracked env
  file for a separate build mode, a local flavor or similar. Leave committed
  config and the Firebase CLI default project (`.firebaserc`) alone, so
  switching back is a restart.
- **Ask before accepting terms.** Steps 2 and 3 accept Firebase and Gemini API
  terms for the lab account. Get the owner's yes first.

## 1. Find the project

Open the Google Cloud console project picker (**All** tab), or run
`CLOUDSDK_ACTIVE_CONFIG_NAME=lab gcloud projects list`. Note the project ID.
It is not a secret.

## 2. Add Firebase to the existing Cloud project

Open `https://console.firebase.google.com/project/<PROJECT_ID>/overview`.

1. Accept the Firebase terms, then read the "few things to remember" page.
   Adding Firebase cannot be undone, which is fine for a disposable project.
2. If the project already has billing, Firebase moves it to the Blaze plan.
   Check that the billing account is the lab's funded one, not a personal one.
3. Turn Google Analytics off unless the test needs it, then **Add Firebase**.

## 3. Turn on AI Logic and register the app

Open **AI Logic → Get started** and pick a provider:

- **Gemini Developer API** — no billing needed.
- **Agent Platform Gemini API** (formerly Vertex AI Gemini API) — needs
  billing, which is what lab credits pay for. The hello world uses the
  `global` location.

Follow the wizard: add a **web** app, give it a nickname and register it without
Hosting. Copy the config with the snippet's copy button and parse `apiKey`,
`appId` and `projectId` into the ignored config. Retyping from a screenshot
invites `O`/`0` and `l`/`I` errors. Then click **Enable APIs**.

**The wizard can spin forever at its App Check step.** In the Cloud console,
**APIs & Services → Enabled APIs & services** must list:

- Firebase AI Logic API (`firebasevertexai.googleapis.com`)
- Agent Platform API (formerly Vertex AI API), for that provider
- Firebase App Check API (`firebaseappcheck.googleapis.com`)

Enable any missing one from its console page, or run
`gcloud services enable firebaseappcheck.googleapis.com --project <PROJECT_ID>`.
Requests work once all three are on, even if the wizard never finishes.

## 4. App Check on localhost: the debug provider

- Serve over a loopback `http://` origin, never `file://`. Keep one origin:
  `localhost:5173` and `127.0.0.1:5173` are different origins with different
  debug tokens.
- Set `self.FIREBASE_APPCHECK_DEBUG_TOKEN = true` before `initializeAppCheck`.
  The SDK creates a token for that browser profile and origin, keeps it in
  IndexedDB and prints it once in DevTools → Console.
- Register the token under **App Check → Apps → ⋮ on the app → Manage debug
  tokens → Add debug token**. The token is a secret, so the human pastes the
  value. If you can read it, for example through the DevTools protocol, copy it
  to the clipboard without printing it, and never commit it.
- Registration is per project. Register the same browser token in each project
  it should reach; one project's registration does not cover another.
- Android, Apple, Flutter and Unity have their own debug providers. Follow the
  App Check debug provider page for that platform.
- **App Check → APIs → Firebase AI Logic** shows the enforcement status and the
  verified share of requests. A finished wizard enforces App Check for AI Logic.
  If the wizard hung, the row can stay at monitoring. Without enforcement the
  web SDK still sends the request after a failed token exchange and the model
  still answers, so a working page proves nothing about App Check. Check that
  `exchangeDebugToken` returns 200 and that verified requests reach 100%.

## 5. Hello world

Copy [`assets/hello-world.html`](assets/hello-world.html) to a scratch folder.
Fill in the three config values and a model the provider serves at that
location, then serve it:

```sh
python3 -m http.server 5173 --bind 127.0.0.1
```

Open `http://127.0.0.1:5173/hello-world.html` in the profile you will keep
testing with. On the first run, register the debug token from the console, then
reload. The page should show `hello world`. For an existing app, apply the same
App Check and AI Logic setup through its ignored lab config.

## 6. Verify with evidence

- In the network log, `exchangeDebugToken` and `generateContent` both return
  200, and both paths contain `/projects/<lab PROJECT_ID>/`. The account the
  console shows decides nothing; the app's config decides which project serves
  and bills the request.
- **App Check → APIs → Firebase AI Logic** shows verified requests at 100%.
- Never record request headers, tokens or raw network exports.

| Symptom | Likely cause | Fix |
|---|---|---|
| `exchangeDebugToken` 403, "App attestation failed" | Token not registered for this project's app | Register it under that app |
| `generateContent` 401, "Firebase App Check token is invalid" | Usually follows the 403 above | Same |
| `generateContent` 403 naming a disabled API | AI Logic or provider API is off | Enable the APIs in step 3 |
| `generateContent` 404 for the model | Model not offered for this provider or location | Pick one from the models page |
| 429 or `RESOURCE_EXHAUSTED` | Lab quota reached | Space out requests; the allowance is finite |

## 7. Before access expires

1. Save prompts, parameters and sanitized results outside the project.
2. Switch the app back by restarting without the lab config. Make one real
   request and confirm the path names the permanent project.
3. Delete the ignored lab config. Run `gcloud auth revoke <lab account>` and
   `gcloud config configurations delete lab`. Close or remove the lab browser
   profile.
4. Make sure nothing that must outlive the account, such as a public demo or
   judging access, depends on the lab project.

## Sources

- [Add Firebase to an existing Google Cloud project](https://firebase.google.com/docs/projects/use-firebase-with-existing-cloud-project)
- [Firebase AI Logic: get started](https://firebase.google.com/docs/ai-logic/get-started) · [models](https://firebase.google.com/docs/ai-logic/models) · [App Check with AI Logic](https://firebase.google.com/docs/ai-logic/app-check)
- [App Check debug provider (web)](https://firebase.google.com/docs/app-check/web/debug-provider) · [enable enforcement](https://firebase.google.com/docs/app-check/enable-enforcement)
- [Firebase API keys are identifiers, not secrets](https://firebase.google.com/docs/projects/api-keys)
- [gcloud named configurations](https://cloud.google.com/sdk/docs/configurations)
