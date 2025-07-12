# Guide: Syncing a GitHub Repository to a GitLab Mirror

This guide outlines the method I'm using for setting up a one-way synchronization from a source repository on GitHub to a mirror on GitLab.

Note that there is no release synchronization, at least not yet. Releases need to be created on both platforms manually.

## Part 1: GitLab Project Setup

### Step 1: Create the GitLab Mirror Project (if not created already)
1.  On GitLab, navigate to **New project > Create blank project**.
2.  Give it a descriptive name (e.g., `my-awesome-bot`), ideally matching the Github Repo, and set its visibility.

### Step 2: Configure Branch Protection
This step is essential to allow the CI/CD job to perform the mirror push, which acts like a force push.

1.  In the new GitLab project, go to **Settings > Repository**.
2.  Expand the **Protected branches** section.
3.  On the rule for the `main` branch, ensure "Allowed to push and merge" is set to at least **Developers + Maintainers**.
4.  Enable the **Allowed to force push** toggle.

### Step 3: Create Project Access Token
This token grants the CI job permission to write to this repository.

1.  In the project, go to **Settings > Access Tokens**.
2.  Click **Add new token**.
3.  **Name:** Give it a clear name, like `CI Mirror Token`.
4.  **Expiration date:** Set it to a date far in the future to avoid frequent renewal.
5.  **Role:** Select **Maintainer**.
6.  **Scopes:** Check the box for **`write_repository`**.
7.  Click **Create project access token**.
8.  **Important:** Immediately copy the generated token string and store it safely.

### Step 4: Add the CI/CD Variable
This securely stores the Access Token for the script to use.

1.  Return to **Settings > CI/CD**.
2.  Expand the **Variables** section and click **Add variable**.
3.  **Key:** `GITLAB_PUSH_TOKEN`
4.  **Value:** Paste the **Project Access Token** you copied in Step 3.
5.  Check both the **"Protect variable"** and **"Mask variable"** boxes.
6.  Click **Add variable**.

### Step 5: Create a Pipeline Trigger
This generates the unique URL that GitHub will call to start the sync process.

1.  Go to **Settings > CI/CD**.
2.  Expand the **Pipeline trigger tokens** section.
3.  Click **Add new token**, give it a description (e.g., `GitHub Sync Trigger`), and click **Create pipeline trigger token**.
4.  **Important:** Immediately copy the generated trigger token string and store it safely.

### Step 6: Create the `.gitlab-ci.yml` File

1.  Create a new file called `.gitlab-ci.yml` with the following contents:

    ```yaml
    # .gitlab-ci.yml

    variables:
      # Change this URL to your source GitHub repository.
      GITHUB_REPO_URL: "https://github.com/YOUR-USERNAME/YOUR-SOURCE-REPO.git"
      GIT_STRATEGY: none

    sync_from_github:
      stage: deploy
      image:
        name: alpine/git:latest
        entrypoint: [""]

      tags:
        - docker

      script:
        - echo "Starting clean-room sync..."
        - git clone --mirror $GITHUB_REPO_URL .
        - git push --mirror "https://gitlab-ci:${GITLAB_PUSH_TOKEN}@${CI_PROJECT_URL#https://}"
        - echo "Sync complete."

      rules:
        - if: '$CI_PIPELINE_SOURCE == "trigger"'
    ```
3.  Replace the placeholder URL following GITHUB_REPO_URL with the correct one for the repository.
4.  Commit and push the file to the `main` branch on GitLab.

## Part 2: GitHub Webhook Setup

### Step 7: Configure the Webhook
1.  Go to your GitHub repository's **Settings > Webhooks**.
2.  Click **Add webhook**.
3.  **Payload URL:** Construct the following URL using the pipeline trigger token from Step 5 and the GitLab project's ID (found under **Settings > General**):

    > `https://gitlab.whiteout-bot.com/api/v4/projects/YOUR_PROJECT_ID/trigger/pipeline?token=YOUR_TRIGGER_TOKEN&ref=main`

4.  **Content type:** Set to `application/json` and leave SSL verification enabled.
5.  **Which events would you like to trigger this webhook?** Select **Just the `push` event**.
6.  Ensure the webhook is **Active** and click **Add webhook**.

---

## You're Done!

The setup is complete. Any push to the GitHub repository will now trigger the GitLab pipeline, which will update the mirror with the latest changes.
