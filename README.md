# SSHFS Sandbox Walkthrough

This walkthrough pairs two Crafting workspaces—**dev** and **ai**—using SSHFS. This setup demonstrates how an AI agent, running in a restricted environment, can monitor a developer's workspace and interact through the filesystem.

The AI workspace mounts the developer's home directory via SSHFS. The "restriction" mode is set to `ALWAYS`, meaning the AI environment stays locked to its predefined access rules, but it can still "see" the developer's work through the mount.

### Communication Flow:
1.  **Developer Commits**: A `post-commit` hook in the `dev` workspace creates a "signal" file (e.g., `.git/review-requested`) in the repository.
2.  **AI Monitors**: The AI workspace, monitoring the mounted `dev` workspace, detects the new file and initiates a (simulated) code review.
3.  **AI Results**: After the review, the AI writes a result file (e.g., `.git/review-status`) back to the mounted `dev` workspace.
4.  **Push Protection**: A `pre-push` hook in the `dev` workspace checks the result file and blocks the push if the AI review has not passed.

---

## Pre-Sandbox Setup

Perform these steps **before** creating the sandbox so the template has the right key material.

1. **Generate the SSH key pair**
   ```bash
   ssh-keygen -t ed25519 -f dev-ai-temp-key -N "" -C "dev-ai-sshfs"
   ```

2. **Create the Crafting secret for the private key**
   ```bash
   # The secret must be --shared so both workspaces can potentially access it
   cs secret create dev-ai-private-key --shared -f dev-ai-temp-key
   ```

3. **Update the template with the public key**
   - Open `dev-ai-temp-key.pub`, copy the full `ssh-ed25519 …` line, and paste it into the `DEV_PUBLIC_KEY` entry in `dev-ai-sshfs.yaml`.

4. **Delete the local private key file**
   ```bash
   rm dev-ai-temp-key
   ```

5. **Create the sandbox**
   ```bash
   cs sandbox create sshfs-ai-demo --from def:dev-ai-sshfs.yaml --wait
   ```

---

## Verify the SSHFS Mount

1. **Open the AI workspace Web IDE**  
   In the Crafting Console, open the Web IDE for the `ai` workspace.
2. **Confirm the mount is present**  
   You should see `dev-workspace` in the Explorer. This is the `dev` workspace's home directory.

---

## Configure the Demo Workflow

In this step, we will set up the local Git hooks in the `dev` workspace that use the filesystem to "talk" to the AI.

1. **Initialize a demo project in the `dev` workspace**
   ```bash
   mkdir -p ~/dev-ai-demo
   cd ~/dev-ai-demo
   git init
   echo "# Demo Project" > README.md
   git add README.md
   git commit -m "Initial commit"
   ```

2. **Add the post-commit "Signal" hook**
   Create `~/dev-ai-demo/.git/hooks/post-commit`:
   ```bash
   #!/bin/bash
   # Signal to the AI that a new commit is ready for review
   echo "pending" > .git/review-status
   echo "AI review requested for commit $(git rev-parse HEAD)"
   ```
   `chmod +x .git/hooks/post-commit`

3. **Add the pre-push "Protection" hook**
   Create `~/dev-ai-demo/.git/hooks/pre-push`:
   ```bash
   #!/bin/bash
   # Block push unless AI review-status is "approved"
   STATUS=$(cat .git/review-status 2>/dev/null || echo "missing")
   if [ "$STATUS" != "approved" ]; then
     echo "Push rejected: AI review status is '$STATUS'. Must be 'approved'."
     exit 1
   fi
   echo "AI review approved. Proceeding with push."
   ```
   `chmod +x .git/hooks/pre-push`

---

## Test the Interaction

1.  **Make a commit in `dev`**
    ```bash
    echo "New feature" >> README.md
    git commit -a -m "Add new feature"
    ```
    The `post-commit` hook will set the status to `pending`.

2.  **Try to push**
    ```bash
    git push origin master  # (Assuming you set up a remote)
    ```
    The `pre-push` hook will block this because the status is `pending`.

3.  **Simulate AI Review (from `ai` workspace)**
    In the `ai` workspace, navigate to the mounted directory and "approve" the commit:
    ```bash
    echo "approved" > ~/dev-workspace/dev-ai-demo/.git/review-status
    ```

4.  **Push again from `dev`**
    ```bash
    git push origin master
    ```
    The push will now succeed!
