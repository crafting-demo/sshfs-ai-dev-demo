# SSHFS Sandbox Walkthrough

This walkthrough pairs two Crafting workspaces—dev and ai—using SSHFS so an AI agent can inspect the developer’s filesystem in real time. The AI workspace mounts the dev home directory, reads the repository contents, writes review feedback, and blocks pushes when feedback is missing. It runs with restriction mode set to “ALWAYS,” meaning the AI environment stays locked to its predefined access rules for the entire sandbox lifetime. The template wires secrets for an SSH key pair: the private key lives in a Crafting secret referenced via ${secret:dev-ai-private-key}, while the public key is injected as plain text.

---

## Pre-Sandbox Setup

Perform these steps **before** creating the sandbox so the template has the right key material.

1. **Generate the SSH key pair**
   ```bash
   ssh-keygen -t ed25519 -f dev-ai-temp-key -N "" -C "dev-ai-sshfs"
   ```

2. **Create the Crafting secret for the private key**
   ```bash
   cs secret create dev-ai-private-key -f dev-ai-temp-key
   ```

3. **Update the template with the public key**
   - Open `dev-ai-temp-key.pub`, copy the full `ssh-ed25519 …` line, and paste it into the `DEV_PUBLIC_KEY` entries in `dev-ai-sshfs.yaml`.

4. **Delete the local private key file**
   ```bash
   rm dev-ai-temp-key
   ```
   Keep the `.pub` file if you want a local record.

With those steps complete, create the sandbox using the updated template.

---

## Verify the SSHFS Mount

1. **Open the AI workspace Web IDE**  
   In the Crafting Console, click the `ai` workspace and choose **Open Web IDE**.

2. **Confirm the mount is present**  
   In the Web IDE, you should see `owner@dev:/home/owner` mounted on `/home/owner/dev-workspace`.

3. **Explore the mounted folder**  
   Use the Explorer sidebar to expand `/home/owner/dev-workspace`. You’re browsing the developer’s home directory directly from the AI workspace.

4. **Create a test file from AI (appears in dev)**  
   - Right-click inside `/home/owner/dev-workspace` and select **New File**.
   - Name it `sshfs-link-check.txt`, open it in the editor, and type:
     ```
     SSHFS test via AI
     ```
   - Save the file (`Ctrl+S`).

5. **Verify from the dev workspace**  
   - Open the Web IDE for the `dev` workspace.
   - In its Explorer, locate `~/sshfs-link-check.txt` and confirm it contains `SSHFS test via AI`.

6. **Clean up (optional)**  
   Delete `sshfs-link-check.txt` from either workspace’s Explorer (or keep it as a marker).

---

## Configure Cross-Workspace Git Hooks

The remaining steps simulate the workflow: a developer commits code in `dev`, the AI workspace analyzes the repo over SSHFS, and a pre-push hook blocks the push until AI feedback is present.

1. **Create the project in the dev workspace**  
   - In the `dev` Web IDE, create `~/dev-ai-demo`.  
   - Open the folder so it appears in the Explorer tree.

2. **Initialize Git**
   ```bash
   cd ~/dev-ai-demo
   git init
   ```

3. **Add initial files**
   - Create `README.md` with:
     ```
     # Dev to AI SSHFS Demo

     Example repo integrating AI workspace checks.
     ```
   - Create `.gitignore` containing:
     ```
     post-commit-AI-review
     ```
   - Stage and commit:
     ```bash
     cd ~/dev-ai-demo
     git add README.md .gitignore
     git commit -m "Initial commit"
     ```

4. **Create the Git hooks**
   - In the Explorer, open `~/dev-ai-demo/.git/hooks` and add:
     - `post-commit`
       ```bash
       #!/bin/bash
       set -euo pipefail

       AI_HOST="${AI_HOST:-ai}"
       AI_SSH_KEY="${HOME}/.ssh/dev-ai"

       ssh -i "${AI_SSH_KEY}" -o BatchMode=yes -o StrictHostKeyChecking=yes owner@"${AI_HOST}" \
         "echo true > ~/post-commit-AI-review"
       ```
     - `pre-push`
       ```bash
       #!/bin/bash
       set -euo pipefail

       AI_HOST="${AI_HOST:-ai}"
       AI_SSH_KEY="${HOME}/.ssh/dev-ai"

       check_file() {
         ssh -i "${AI_SSH_KEY}" -o BatchMode=yes -o StrictHostKeyChecking=yes owner@"${AI_HOST}" "$@"
       }

       if ! check_file 'test -f ~/post-commit-AI-review'; then
         printf "AI review marker missing in AI workspace. Run a commit to trigger AI review.\n" >&2
         exit 1
       fi

       REVIEW_CONTENT="$(check_file 'cat ~/post-commit-AI-review' | tr -d '\r\n')"

       if [[ "${REVIEW_CONTENT}" != "true" ]]; then
         printf "AI review marker must contain \"true\" (found \"%s\"). Resolve before pushing.\n" "${REVIEW_CONTENT}" >&2
         exit 1
       fi

       exit 0
       ```
   - In the terminal:
     ```bash
     chmod +x ~/dev-ai-demo/.git/hooks/post-commit
     chmod +x ~/dev-ai-demo/.git/hooks/pre-push
     ```

5. **Test the post-commit hook**
   - Append a line (for example, “Another line”) to `README.md` in the dev workspace.
   - Commit:
     ```bash
     cd ~/dev-ai-demo
     git add README.md
     git commit -m "Run post-commit hook"
     ```
   - In the AI workspace Explorer, open `~/post-commit-AI-review` and confirm it now contains `true` (the notional “AI approval”).

6. **Test the pre-push hook**
   - Create a bare repo to act as an origin:
     ```bash
     mkdir -p ~/dev-ai-demo-remote.git
     cd ~/dev-ai-demo-remote.git && git init --bare
     cd ~/dev-ai-demo && git remote add origin ~/dev-ai-demo-remote.git
     ```
   - In the AI workspace terminal, simulate a failed review:
     ```bash
     echo false > ~/post-commit-AI-review
     ```
   - Back in the dev terminal, attempt a push:
     ```bash
     cd ~/dev-ai-demo
     git push origin master
     ```
     The push should be rejected because the AI review marker is not `true`.
   - Reset the marker in the AI workspace:
     ```bash
     echo true > ~/post-commit-AI-review
     ```
   - Push again from the dev workspace:
     ```bash
     cd ~/dev-ai-demo
     git push origin master
     ```
     The push now succeeds.