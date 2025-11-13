# Publishing from CI with 2FA

Publishing from CI needs some extra setup to stay safe and avoid unwanted publishes. This is meant for teams or projects that release often and have more than one maintainer.

This guide shows how to use **GitHub Environments** together with **npm registry 2FA**. Both are important for keeping your package secure.

- **GitHub Environments**: Keep your publish token in an environment, not as a normal repo secret. Make sure the workflow only publishes (no install). This helps prevent token leaks from simple CI mistakes.
- **npm registry 2FA**: Adds an extra check so no one can publish without your one-time password.

If you haven’t read the [Publishing Locally](https://github.com/npm-pub-2025/local-publish) guide yet, do that first. The steps there for npm account security are always required.


## TL;DR

The most important security steps are:

- Create a GitHub Environment for publishing
- Add required reviewers for approval to help prevent accidental or unauthorized releases
- Use a Granular Access Token (GAT) with 2FA required
- Store your token as a GitHub Environment secret
- Set up a workflow with permissions and OTP input
- Approve and provide your OTP when publishing. This step depends on a trusted third-party service


## Prerequisites

Before starting, make sure you have:
- A GitHub repository with Actions enabled
- An npm account with 2FA turned on
- A trusted team for approvals


## Setup Steps

### 1. Create an Environment

Go to **Settings → Environments** in your GitHub repo and click **New Environment**.

![New Environment](assets/new-env.png)

Give it a name.

![Name your Environment](assets/name-env.png)

### 2. Add Reviewers

Add **required reviewers**. This means your publish workflow will need approval before it runs. You can allow self-approval, but that’s less secure.

![Required reviewers](assets/required-reviewers-env.png)

### 3. Make a Granular Access Token (GAT)

In your npm account, go to **Access Tokens** and create a new **Granular Access Token**.

- Keep *“Bypass 2FA”* unchecked, if it is checked (the default for tokens created before Nov 5th 2025) be sure to uncheck it
- Set an expiration date you think is reasonable
- Give it read/write access only to your package

![GAT Scope](assets/gat-scope.png)

### 4. Save the Token in GitHub

In your GitHub Environment, add a **secret** with your GAT. You’ll use this in your workflow.

![GAT Secret](assets/secret-env.png)

### 5. Set Up the Workflow

You can check the full example [here](.github/workflows/publish.yml), but these are the main parts.

#### Permissions

```yaml
permissions:
  contents: write
  id-token: write
```

These let the workflow push commits and use secure tokens.

#### Environment

```yaml
environment: publish
```

This connects the workflow to the Environment you made.

#### 2FA Setup

Use this step to handle your npm one-time password:

```yaml
- uses: step-security/wait-for-secrets@v1
  id: otp
  with:
    slack-webhook-url: ${{ secrets.SLACK_WEBHOOK_URL }}
    secrets: |
      OTP:
        name: 'npm otp'
        description: 'An npm one time password'
```

Then use it in the publish command:

```yaml
- name: npm publish
  run: |
    npm publish --access=public --tag="${{ inputs.tag }}" --otp="${{ steps.otp.outputs.OTP }}"
```

## How to Publish

This workflow uses a manual `workflow_dispatch` trigger. You can change it later if needed.

To publish:
1. Go to **Actions → Publish** in GitHub
2. Click **Run workflow**

![Run workflow](assets/run-workflow.png)

Your teammates will get an approval request email.

![Review email](assets/review-email.png)

Once approved, the workflow starts. When it reaches the `wait-for-secrets` step, it will pause and show a link for you to enter your OTP.

![wait-for-secrets](assets/wait-for-secrets.png)

Go to the link, enter your OTP, and the publish will finish.

![Provide OTP](assets/otp.png)

## This Example is Opinionated

This setup reflects one way of publishing safely from CI. You can change details to fit your project’s needs. For example:
- You might skip reviewers for small teams or solo maintainers
- You can use a different OTP action if you prefer
- The publish order (npm first or GitHub first) is up to you, use the appropriate trigger for your workflow (just never `pull_request_target`)
- For extra security, consider using [harden-runner](https://github.com/step-security/harden-runner) or a similar endpoint protection tool for GitHub Actions runners. It helps control egress traffic and adds useful safeguards to your CI pipeline


## Tips

- Use CI publishing only when you need it. Local publishing is simpler and safer for most projects
- Check your GitHub Actions dependencies often. Specifically ensure any [pinned to commits are not from a fork](https://www.chainguard.dev/unchained/what-the-fork-imposter-commits-in-github-actions-and-ci-cd)

**Multi-Releasers Strategy**

When multiple team members are responsible for publishing releases, each person should create their own personal npm token before running a release.

They should then:
1. Temporarily add their token to the CI environment or GitHub secrets to perform the release
2. Remove the token from both npm and the CI/GitHub secrets immediately after the release to maintain security

This approach is more secure and auditable than using a shared bot account with publish permissions, as it provides clear attribution in security logs and incident investigations.

Note: npm requires tokens to be rotated every 90 days, so this practice also aligns with npm’s security policies.