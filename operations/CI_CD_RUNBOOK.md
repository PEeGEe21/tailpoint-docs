# GitHub CI/CD Runbook

Tailpoint's frontend, backend, and admin are separate GitHub repositories. Each repository now has:

- `CI`: runs for every pull request and pushes to `main` or `dev`.
- `Deploy production`: waits for a successful `CI` run on `main`, then triggers the configured hosting provider. It can also be started manually.

## Repository setup

Repeat these steps in all three GitHub repositories:

1. Create a GitHub environment named `production`.
2. Add an environment secret named `DEPLOY_HOOK_URL` containing that application's hosting-provider deploy hook.
3. Add required reviewers to the `production` environment if production deploys need manual approval.
4. Protect `main`: require a pull request, require the repository's CI check, require the branch to be current, and block force pushes.
5. Do not add environment values or application credentials to the deploy hook URL or workflow files.

The three `DEPLOY_HOOK_URL` values should be different. A frontend merge must not deploy the backend or admin application.

## Release and rollback

Merge finished work into `main`; CI verifies the commit and a successful run triggers the production workflow. Check the hosting provider's deployment page for completion because a successful hook call only confirms that the provider accepted the request.

For rollback, use the hosting provider's rollback/redeploy function for the last known-good artifact. If the provider cannot roll back artifacts, revert the faulty commit through a pull request; merging the revert triggers a new deployment.

## Current validation commands

- Frontend: `npm ci`, `npm run build`
- Backend: `npm ci`, `npm run typecheck`, `npm run test:smoke -- --ci`, `npm run build`
- Admin: `npm ci`, advisory `npm run lint`, blocking `npm run typecheck`, `npm run build`

Admin lint is temporarily advisory because the repository had 28 existing lint errors when CI was introduced. Remove `continue-on-error` from its workflow after that backlog is cleared.

Frontend CI uses reserved `.invalid` URLs for required public variables. They are build-only placeholders and cannot receive traffic. Configure the real `NEXT_PUBLIC_*` production values in Vercel; do not put production values in the workflow.

Backend CI similarly uses isolated placeholder configuration so unit tests can import the validated application config. The smoke tests mock their service dependencies and do not connect to the placeholder database or storage services. Production credentials belong in Render, not in GitHub workflow source.

### Backend AI production configuration

The backend deployment workflow validates AI release configuration before it calls the Render deploy hook. Configure these as GitHub `production` environment variables:

- `AI_TEXT_PROVIDER` (`none` until the pilot, then `openai`)
- `AI_TRANSCRIPTION_PROVIDER` (`none` or `openai`)
- `TRANSCRIPTION_ENABLED`
- `OPENAI_AI_MODEL`
- `OPENAI_TRANSCRIPTION_MODEL`
- `HUGGINGFACE_AI_MODEL`
- `AI_PROVIDER_TIMEOUT_MS`
- `AI_USER_HOURLY_LIMIT`
- `AI_ORGANIZATION_HOURLY_LIMIT`
- `AI_INPUT_COST_PER_MILLION`
- `AI_OUTPUT_COST_PER_MILLION`

Add `OPENAI_API_KEY` as a GitHub `production` environment secret. The workflow fails closed if an OpenAI provider is selected without it.

For Hugging Face routed inference, set `AI_TEXT_PROVIDER=huggingface` and add `HF_TOKEN` as a GitHub `production` environment secret. Mirror both values in Render. Hosted Hugging Face inference includes only small monthly experimentation credits and becomes paid after those credits are exhausted; it is not an unlimited free production service.

The deploy hook does not transfer these values to Render. Mirror the same variables and `OPENAI_API_KEY` in the Render backend service environment; Render is the runtime source of truth. Keep providers set to `none` until the pilot organization is ready.

The frontend build receives a 4 GB Node heap and persists `.next/cache` between runs. Vercel's Git auto-deploy must be disabled if GitHub Actions is intended to be the production release gate; otherwise Vercel starts its own build immediately on every push, independently of CI.




`DEPLOY_HOOK_URL` comes from whichever platform hosts that specific application.

## Getting the deploy hook

### Vercel

For each Vercel project:

1. Open the project in Vercel.
2. Go to **Settings → Git**.
3. Find **Deploy Hooks**.
4. Enter a name such as `GitHub Production`.
5. Select the production branch: `main`.
6. Create the hook and copy its URL.

Vercel deploy hooks accept the `POST` request used by our workflow. Treat the URL like a password. [Vercel deploy-hook documentation](https://vercel.com/docs/deploy-hooks)

### Render

For each Render service:

1. Open the service in Render.
2. Go to **Settings**.
3. Find **Deploy Hook**.
4. Copy the hook URL.

Render accepts either `GET` or `POST`, so our workflow is compatible. [Render deploy-hook documentation](https://render.com/docs/deploy-hooks)

### Which provider should each app use?

A sensible setup is:

| Application | Likely host |
|---|---|
| `track-a-project` frontend | Vercel |
| `tracker-admin` frontend | Vercel |
| `track-a-project-backend` | Render |

You therefore need three different URLs:

- Frontend Vercel deploy hook
- Admin Vercel deploy hook
- Backend Render deploy hook

Do not reuse one URL between repositories.

## Adding the URL to GitHub

Repeat this inside each GitHub repository:

1. Open the repository.
2. Go to **Settings → Environments**.
3. Select **New environment**.
4. Name it exactly `production`.
5. Open the new environment.
6. Under **Environment secrets**, select **Add secret**.
7. Name it exactly:

```text
DEPLOY_HOOK_URL
```

8. Paste that repository’s Vercel or Render hook URL.
9. Save it.

Optionally add **Required reviewers** to the environment. That makes GitHub pause before each production deployment until you approve it.

If Render or Vercel already deploys automatically whenever `main` changes, disable that platform’s automatic deployment. Otherwise, merging into `main` can cause two deployments: one from the Git integration and another from our hook. Render documents its automatic deployment behavior [here](https://render.com/docs/deploys).

## Protecting `main`

First, push the workflow and open at least one PR so GitHub runs CI once. GitHub may not offer the check in its selection list until it has run.

Then repeat this in all three repositories:

1. Open **Settings → Branches**.
2. Under **Branch protection rules**, select **Add rule**.
3. Enter this branch name pattern:

```text
main
```

4. Enable **Require a pull request before merging**.
5. Enable **Require status checks to pass before merging**.
6. Enable **Require branches to be up to date before merging**.
7. Search for and select the CI job:

| Repository | Expected check |
|---|---|
| Frontend | `build` |
| Backend | `verify` |
| Admin | `verify` |

8. Enable **Require conversation resolution before merging**.
9. Leave **Allow force pushes** disabled.
10. Leave **Allow deletions** disabled.
11. If available, enable **Do not allow bypassing the above settings**.
12. Save changes.

GitHub requires selected checks to succeed, skip, or finish neutral before a protected branch can be updated. [GitHub branch-protection documentation](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches)

One important detail: the admin lint step is currently advisory, but `verify` still requires its type-check and production build to pass.
