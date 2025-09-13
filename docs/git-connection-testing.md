# Git Connection Testing – Test Solution

This document describes a practical, repeatable way to verify and troubleshoot Git connectivity from a local machine (including VS Code) to remote Git hosting providers (e.g., GitHub, GitLab, Bitbucket). It covers both HTTPS and SSH, offers clear pass/fail criteria, and includes Windows/PowerShell specifics.

## Scope
- Validate local Git installation and identity.
- Confirm network reachability to common Git endpoints.
- Test HTTPS auth flows (incl. Personal Access Token).
- Test SSH auth flows (keys, agent, host keys).
- Verify repository remotes and basic operations (fetch, clone, push).
- Provide troubleshooting guidance and success criteria.

## Prerequisites
- Git installed: https://git-scm.com/downloads
- Windows OpenSSH Client (Windows 10/11 usually includes it). If needed, add via “Optional Features”.
- Access to your Git provider account and an existing repository (or permission to create one).
- For HTTPS: ability to use a Personal Access Token (PAT) when prompted.
- For SSH: permission to add a public key to your account.

## Quick Start (Happy Path)
1. Verify Git and config:
   ```powershell
   git --version
   git config --global user.name
   git config --global user.email
   ```
2. Check remote(s) from an existing repo folder (if you have one):
   ```powershell
   git remote -v
   git ls-remote origin
   ```
3. Network smoke test:
   ```powershell
   # SSH port (22) and HTTPS (443)
   Test-NetConnection github.com -Port 22
   Test-NetConnection github.com -Port 443
   ```
4. HTTPS test (read-only):
   ```powershell
   git ls-remote https://github.com/<org>/<repo>.git
   ```
   - Pass: References listed with no error (may prompt for credentials).
5. SSH test (handshake):
   ```powershell
   ssh -T git@github.com
   ```
   - Pass: Message like “Hi <user>! You've successfully authenticated, but GitHub does not provide shell access.”

## Detailed Test Cases

### T1. Environment & Identity
- Commands:
  ```powershell
  git --version
  git config --global --list
  ```
- Pass: Git version prints; global name/email configured.
- Fix: Set identity if missing:
  ```powershell
  git config --global user.name "Your Name"
  git config --global user.email "you@example.com"
  ```

### T2. Network Reachability
- Commands:
  ```powershell
  Test-NetConnection github.com -Port 443
  Test-NetConnection github.com -Port 22
  # Optional verbose SSH check
  ssh -vT git@github.com
  ```
- Pass: `TcpTestSucceeded: True` for required ports; SSH handshake reaches provider.
- Fix: Check VPN/proxy/firewall; allow outbound 22 (SSH) and 443 (HTTPS). If 22 blocked, prefer HTTPS or SSH-over-HTTPS if provider supports it.

### T3. HTTPS Connectivity
- Read-only check (no clone yet):
  ```powershell
  git ls-remote https://github.com/<org>/<repo>.git
  ```
- Clone test:
  ```powershell
  git clone https://github.com/<org>/<repo>.git
  ```
- Push test (optional, needs write access):
  ```powershell
  cd <repo>
  git checkout -b connection-test
  echo "hello" > connectivity_check.txt
  git add connectivity_check.txt
  git commit -m "chore: connectivity check"
  git push -u origin connection-test
  ```
- Notes:
  - When prompted for password, use a Personal Access Token (PAT) if provider requires.
  - Consider enabling a credential helper (Windows installs Git Credential Manager by default).

### T4. SSH Connectivity
1. Check for existing keys:
   ```powershell
   dir $env:USERPROFILE\.ssh
   ```
   Look for `id_ed25519` and `id_ed25519.pub` (preferred) or `id_rsa`/`.pub`.

2. Generate a key if needed:
   ```powershell
   ssh-keygen -t ed25519 -C "you@example.com"
   # Press Enter to accept default path: C:\Users\<you>\.ssh\id_ed25519
   # Optionally set a passphrase.
   ```

3. Start SSH agent and add key:
   ```powershell
   Get-Service ssh-agent | Set-Service -StartupType Automatic
   Start-Service ssh-agent
   ssh-add $env:USERPROFILE\.ssh\id_ed25519
   ```

4. Add public key to provider:
   - Copy the public key contents:
     ```powershell
     Get-Content $env:USERPROFILE\.ssh\id_ed25519.pub
     ```
   - Paste into your Git provider’s SSH keys settings.

5. Verify SSH handshake:
   ```powershell
   ssh -T git@github.com
   # For others:
   # ssh -T git@gitlab.com
   # ssh -T git@bitbucket.org
   ```

6. Switch an existing remote to SSH (optional):
   ```powershell
   git remote set-url origin git@github.com:<org>/<repo>.git
   git remote -v
   ```

- Pass: Successful greeting/auth message; `git fetch` works without credential prompts.

### T5. Repository Operations
- Fetch:
  ```powershell
  git fetch origin --prune
  ```
- Pull:
  ```powershell
  git pull --rebase
  ```
- Push (requires write):
  ```powershell
  git push
  ```
- Pass: Commands complete without auth errors; expected refs updated.

### T6. VS Code Integration (Optional)
- Open the folder in VS Code and check the Source Control view.
- Verify that pulling/pushing via the UI works and respects the configured protocol (HTTPS/SSH).
- If VS Code prompts for credentials, ensure they match the method you validated above.

## Troubleshooting Guide
- Permission denied (publickey): Ensure the correct private key is loaded into `ssh-agent` and that the matching public key is added to your account.
- Host key verification failed: Remove outdated entry and retry to accept the new host key.
  ```powershell
  notepad $env:USERPROFILE\.ssh\known_hosts
  # Remove the line for the host, save, then retry SSH.
  ```
- Wrong remote URL: Inspect and correct with `git remote -v` and `git remote set-url`.
- Corporate proxy blocks SSH (22): Use HTTPS remotes, or configure SSH over port 443 if your provider supports it.
- Credential prompts loop on HTTPS: Clear cached creds and retry with PAT.
  ```powershell
  git credential-manager reject https://github.com
  ```
- Multiple Git installs or shells: Prefer a single environment (PowerShell or Git Bash) and ensure `where git` points to the expected binary.

## Success Criteria (Pass/Fail)
- Pass:
  - Network checks succeed for chosen protocol.
  - HTTPS: `git ls-remote` and `git clone` succeed; push works with PAT if permitted.
  - SSH: `ssh -T` returns successful auth message; `git fetch/push` succeed without extra prompts.
- Fail:
  - Any step above errors with connectivity/auth failures that cannot be resolved via troubleshooting.

## Appendix A: Minimal Verification Script (Manual Steps)
Use these commands in a clean folder to minimally verify end-to-end:
```powershell
# Replace <org>/<repo>
$repo = "https://github.com/<org>/<repo>.git"

# HTTPS read test
git ls-remote $repo

# Optional: clone and fetch
mkdir test-https; cd test-https
git clone $repo .
git fetch --prune
cd ..

# SSH handshake
ssh -T git@github.com
```

## Appendix B: Switching Protocols Safely
- From HTTPS to SSH:
  ```powershell
  git remote set-url origin git@github.com:<org>/<repo>.git
  ```
- From SSH to HTTPS:
  ```powershell
  git remote set-url origin https://github.com/<org>/<repo>.git
  ```
- Verify:
  ```powershell
  git remote -v
  ```

---
If you want this tailored to a specific provider, organization, or repository, replace placeholders above and/or request a provider-specific edition.
