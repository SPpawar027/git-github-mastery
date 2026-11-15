# Chapter 06: Authentication Troubleshooting Field Guide (The Diagnostic Battle-Card)

---

## 1. Concept: Ye Kya Hai?

Authentication errors software development ke sabse frustrating roadblocks hote hain. 
Aapka code 100% ready hai, tests pass hain, release deadline sar par hai, lekin terminal chillata hai:
`Permission denied (publickey)` ya `fatal: Authentication failed`.

Ek amateur developer random commands copy-paste karke SSH keys delete kar deta hai aur system ko aur corrupt kar leta hai.
Ek **Senior Technical Lead** systematic **Forensic Tree** follow karta hai:
$$\text{Network Layer} \longrightarrow \text{Host Verification} \longrightarrow \text{Credential Presentation} \longrightarrow \text{Organization Authorization (SAML SSO)}$$

Ye chapter aapka universal **Diagnostic Field Guide** hai jo har authentication error ko 60 seconds ke andar diagnose aur resolve karne ki methodology deta hai.

---

## 2. Why: Iski Zarurat Kyu Padi? (The 7 Major Auth Errors)

Real-world production environments mein 99% authentication failures in 7 categories mein aate hain:
1. `Permission denied (publickey)`
2. `Support for password authentication was removed on August 13, 2021`
3. `ssh: connect to host github.com port 22: Connection timed out`
4. `Host key verification failed`
5. `ERROR: Repository not found` (The 404 Privacy Mask)
6. `fatal: could not read Username for 'https://github.com': No such device or address`
7. `Resource protected by organization SAML enforcement`

---

## 3. Real-World Example: The "Repository Not Found" Privacy Mask

Aap ek naye client ke project par kaam karne ke liye hire huye.
Client ne aapse bola: *"Maine aapko hamari private repository `client-firm/billing-api` mein collaborator add kar diya hai!"*

Aapne terminal par chalaya:
`git clone git@github.com:client-firm/billing-api.git`

Terminal ne error throw kiya:
`ERROR: Repository not found.`
`fatal: Could not read from remote repository.`

Aap confuse ho gaye: *"Client ne repo link galat diya hai kya? Ye bol raha hai repo exist hi nahi karti!"*

### The Cybersecurity Reason:
GitHub security policy ke tehat **Private Repositories ke existence ko unauthenticated users se chupata hai!**
Agar GitHub bolta: *"Repository exist karti hai lekin aapke paas access nahi hai"*, toh attackers private repo names scan kar lete!
Isiliye GitHub 403 Forbidden ke bajaye direct **404 Not Found** mask lagata hai. Is error ka 90% matlab hota hai: **"Aapki authentication credential invalid hai ya aapke account ne invitation accept nahi kiya hai!"**

---

## 4. Internal Working: `~/.ssh/known_hosts` & Host Key Fingerprints

Jab aap pehli baar kisi server se SSH connect karte hain, terminal poochta hai:
`The authenticity of host 'github.com (140.82.121.4)' can't be established. Are you sure you want to continue connecting (yes/no)?`

### Why this happens:
Ye **Man-in-the-Middle (MITM) Protection** hai!
Aapka computer verify kar raha hai ki kya aap sach mein official GitHub server se baat kar rahe hain ya kisi hacker ke fake proxy server se.
Jab aap `yes` type karte hain, GitHub ka cryptographic public host key aapke computer ke `~/.ssh/known_hosts` file mein permanently freeze ho jata hai.

### Official GitHub Public Key Fingerprints:
- **Ed25519:** `SHA256:+DiY3wvvV6TuJJhbpZisF/zLDA0zPMSvHdkr4UvCOqU`
- **ECDSA:** `SHA256:p2QAMXNIC1TJYWeIOttrVc98/R1BUFWu3/LiyKgUfQM`
- **RSA:** `SHA256:uNiVztksCsDhcc0u9e8BujQXVUpKZIDTMczCvjBWDgY`

Agar kabhi terminal bole `Host key verification failed!`, iska matlab ya toh network par koi attack ho raha hai, ya GitHub ne official host keys rotate ki hain!

---

## 5. The Master Diagnostic Battle-Card

| Error Message | Root Cause | Instant Fix Command |
|---|---|---|
| `Permission denied (publickey)` | SSH key loaded nahi hai ya GitHub par uploaded nahi hai. | `ssh-add ~/.ssh/id_ed25519` aur check `ssh -T git@github.com` |
| `Support for password authentication was removed` | HTTPS clone par web password use kiya ja raha hai. | SSH par switch karein: `git remote set-url origin git@github.com:...` |
| `port 22: Connection timed out` | Office / College Wi-Fi firewall port 22 ko block kar raha hai. | SSH over Port 443 configure karein in `~/.ssh/config` |
| `Host key verification failed` | `~/.ssh/known_hosts` mein purani outdated host key cached hai. | `ssh-keygen -R github.com` aur fresh keys fetch karein |
| `Repository not found` | Private repo invitation pending hai ya wrong SSH key active hai. | GitHub repo invitation accept karein aur `ssh -vT` se active user check karein |
| `Resource protected by organization SAML enforcement` | Company organization ne SSO mandatory kiya hai, key authorized nahi hai. | GitHub Settings -> SSH Keys -> Click **"Configure SSO"** -> Authorize! |

---

## 6. Expected Output: Live Forensic Debugging Traces

Jab authentication fail ho, toh deep forensic debug run karein:

```bash
$ ssh -vT git@github.com
OpenSSH_9.6p1, OpenSSL 3.0.13 30 Jan 2024
debug1: Connecting to github.com [140.82.121.4] port 22.
debug1: Connection established.
debug1: Host 'github.com' is known and matches the ED25519 host key.
debug1: Will attempt key: /home/user/.ssh/id_ed25519 ED25519 SHA256:4f8a...
debug1: Offering public key: /home/user/.ssh/id_ed25519 ED25519
debug1: Server accepts key: /home/user/.ssh/id_ed25519 ED25519
debug1: Authentication succeeded (publickey).
Authenticated to github.com ([140.82.121.4]:22) using "publickey".
Hi alice-engineer! You've successfully authenticated...
```

Notice: `ssh -vT` exact reason bata deta hai ki connection kis line par atak raha hai!

---

## 7. Visual Explanation: The 60-Second Authentication Decision Tree

```text
                               ┌────────────────────────────────┐
                               │   AUTHENTICATION FAILS (PUSH)  │
                               └───────────────┬────────────────┘
                                               │
                       ┌───────────────────────┴───────────────────────┐
                       ▼                                               ▼
               Protocol is HTTPS?                              Protocol is SSH?
                       │                                               │
          ┌────────────┴────────────┐                     ┌────────────┴────────────┐
          ▼                         ▼                     ▼                         ▼
   Password Error?           403 / 404 Token?      Port 22 Timeout?          Permission Denied?
   (Deprecation)             (Scope / Expiry)      (Firewall Block)          (Missing Public Key)
          │                         │                     │                         │
          ▼                         ▼                     ▼                         ▼
   Switch to SSH!            Generate new PAT      Route over Port 443       Run: ssh-add -l &
   (git remote set-url)      with 'repo' scope.    (ssh.github.com:443)      Upload to GitHub!
```

---

## 8. Common Mistakes: Developers Kya Galti Karte Hain?

1. **Blindly Deleting `~/.ssh/known_hosts`:**
   - Mistake: Host key warning aane par poori `known_hosts` file delete kar dena.
   - Consequence: Saare AWS servers, staging droplets, aur servers ke host fingerprints wipe out ho jaate hain. Always use surgical command: `ssh-keygen -R github.com`!
2. **Missing `repo` Scope on Personal Access Tokens:**
   - Developer HTTPS token generate karta hai lekin scopes select karte waqt **`repo` (Full control of private repositories)** tick karna bhool jata hai. Token valid hone ke baad bhi clone/push 404 error deta hai.
3. **Ignoring Organization SAML SSO Authorization:**
   - Corporate GitHub organization mein join hone ke baad apni existing personal SSH key se push karna bina organization SSO authorize kiye.

---

## 9. Troubleshooting Guide: The 7-Step Diagnostic Framework

Agar Organization SAML SSO error aaye:

```text
1. OBSERVE:    Command: `git push origin feature/auth`
               Error: "ERROR: The `enterprise-corp` organization has enabled SAML SSO.
                      You must authorize your SSH key to access this organization."

2. INSPECT:    Identify the fingerprint of your active key:
               `ssh-add -l`

3. UNDERSTAND: SSH key valid hai aur personal repos par kaam kar rahi hai,
               lekin corporate organization ne strict identity policy lagayi hai
               jo mandate karti hai ki employee corporate SSO (Okta/Google) se key link kare.

4. PLAN:       GitHub Web UI par jakar key ko corporate organization ke liye
               explicitly authorize karein.

5. EXECUTE:    Step 1: Go to https://github.com/settings/keys
               Step 2: Find your active SSH key.
               Step 3: Click "Configure SSO" dropdown next to the key.
               Step 4: Click "Authorize" next to `enterprise-corp` organization.
               Step 5: Login with your corporate Okta/Google credentials in browser.

6. VERIFY:     Run `git push origin feature/auth` again in terminal;
               push instantly succeeds!

7. PREVENT:    Naye laptop par SSH key upload karte hi turant "Configure SSO"
               authorize karne ki aadat daalein.
```

---

## 10. Practice Lab: Simulating & Fixing Known_Hosts & Port 443 Fallbacks 💻

Chaliye apne terminal par authentic host verification aur port 443 SSH fallback inspect karte hain:

```bash
# Step 1: Query GitHub's official Ed25519 public host key live from DNS:
ssh-keyscan -t ed25519 github.com 2>/dev/null
# Notice the output: github.com ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIOMqqnkVzrm0SdG6UOoqKLsabgH5C9okWi0dh2l9GKJl

# Step 2: Test SSH connection directly over Port 443 (The Firewall Bypass Test!)
ssh -T -p 443 git@ssh.github.com
# Notice: Successful authentication even over HTTPS port 443!

# Step 3: Run verbose debug trace to see the exact cipher negotiation
ssh -vT git@github.com 2>&1 | grep -E "Authentications|Offering|Server accepts"
# Output clearly shows:
# debug1: Authentications that can continue: publickey
# debug1: Offering public key...
# debug1: Server accepts key...

# Step 4: Check if GitHub CLI is installed and check its auth status:
if command -v gh &> /dev/null; then
    gh auth status
else
    echo "GitHub CLI (gh) not installed. Native Git SSH is fully operational!"
fi
```

---

## 11. Challenge: Test Your Understanding 🧠 (Scenario-Based Problems)

### 📌 Scenario Challenge 1: The "Host Key Verification Failed" Incident
> **Problem Statement:**
> March 2023 mein GitHub ne apna RSA SSH Host Key rotate (change) kiya tha kyunki ek security audit mein purani key brief period ke liye internal logs mein expose ho gayi thi.
> Subah aate hi duniya ke hazaron developers ke terminal crash hone lage error ke sath:
> `@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@`
> `@    WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!     @`
> `@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@`
> `IT IS POSSIBLE THAT SOMEONE IS DOING SOMETHING NASTY!`
> `Someone could be eavesdropping on you right now (man-in-the-middle attack)!`
> `Host key verification failed.`
> `fatal: Could not read from remote repository.`
> 
> Junior developers panic karke bolne lage: *"GitHub hack ho gaya hai!"*
> 
> **Questions:**
> 1. OpenSSH client ne connection turant terminate kyu kiya?
> 2. Ek Senior Lead official GitHub security advisory verify karke purane cached host key ko `~/.ssh/known_hosts` se safely delete aur naye verified key se replace kaise karta hai?

<details>
<summary><b>💡 Click to Reveal Complete Solution & Walkthrough</b></summary>

#### Solution & Forensic Resolution:

**1. Why OpenSSH aborted:**
`~/.ssh/known_hosts` file mein purani RSA host key ka fingerprint saved tha.
Jab GitHub ke server ne nayi rotated key present ki, fingerprint mismatch ho gaya!
OpenSSH ne security rule trigger kiya: *"Ye Man-in-the-Middle attack ho sakta hai! Connection instantly freeze karo!"*

#### 2. The Senior Lead Remediation Steps:

**Step A: Remove only the stale GitHub host key:**
```bash
ssh-keygen -R github.com
```

**Step B: Fetch official verified new key from GitHub:**
GitHub ke official documentation par jakar official SHA-256 fingerprint verify karein:
`SHA256:uNiVztksCsDhcc0u9e8BujQXVUpKZIDTMczCvjBWDgY` (RSA)
`SHA256:+DiY3wvvV6TuJJhbpZisF/zLDA0zPMSvHdkr4UvCOqU` (Ed25519)

**Step C: Connect and accept the new verified host key:**
```bash
ssh -T git@github.com
# Type "yes" when the verified fingerprint matches!
```
Connection permanently restore ho gaya!
</details>

---

### 📌 Scenario Challenge 2: Headless Docker Container HTTPS Push Error
> **Problem Statement:**
> Ek automated CI Docker container headless environment mein chal raha hai (no GUI, no interactive TTY).
> Build script ke andar line execute hoti hai:
> `git push https://github.com/my-org/my-repo.git main`
> 
> Container turant crash ho jata hai error dekar:
> `fatal: could not read Username for 'https://github.com': No such device or address`
> 
> **Question:** Headless non-interactive environment mein bina interactive prompt ke GitHub token authenticate karne ki standard environment variable injection technique kya hai?

<details>
<summary><b>💡 Click to Reveal Complete Solution & Walkthrough</b></summary>

#### Solution: URL Token Injection via CI Secrets

Headless container ke paas koi interactive terminal TTY nahi hota jahan wo aapse username pooch sake.

#### The Production Solution:
CI runner ke secure environment secrets se token read karke Git URL ke andar inject karein:

```bash
git push https://x-access-token:${GITHUB_TOKEN}@github.com/my-org/my-repo.git main
```

#### Why `x-access-token`:
GitHub standard convention follow karta hai:
- Username: `x-access-token` (ya `oauth2`)
- Password: `${GITHUB_TOKEN}` (Secret token injected from CI vault)
Terminal prompt completely bypass ho jata hai aur automated push 100% succeed hota hai!
</details>

---

## 12. Interview Questions (FAANG & Product Companies)

### Q1: Why does GitHub return `fatal: repository not found` instead of `403 Permission Denied` when an unauthorized user attempts to access a private repository?
**Answer:** This is a deliberate defense-in-depth security measure known as **Privacy Masking**. If GitHub returned `403 Forbidden`, it would confirm to an attacker or unauthorized entity that a private repository with that specific organization and repository name actually exists on GitHub's infrastructure. By returning `404 Not Found`, GitHub completely hides the existence of private enterprise repositories, preventing automated name reconnaissance and dictionary enumeration attacks.

### Q2: What causes `Host key verification failed` in SSH and why should developers avoid disabling `StrictHostKeyChecking`?
**Answer:** `Host key verification failed` occurs when the remote server's cryptographic public host key does not match the fingerprint previously recorded in `~/.ssh/known_hosts`. Disabling `StrictHostKeyChecking=no` makes the SSH client vulnerable to **Man-in-the-Middle (MITM) attacks**, where an attacker on the same local network or a compromised DNS server intercepts the connection, decrypts the session traffic, and captures credentials without detection.

### Q3: What is GitHub SAML SSO enforcement for SSH keys?
**Answer:** In GitHub Enterprise Organizations integrated with corporate Identity Providers (IdPs like Okta, Ping Identity, or Azure AD), SAML Single Sign-On enforcement mandates that any SSH key or Personal Access Token must be explicitly authorized against the organization's SAML identity before it can access organization repositories. Even if an SSH key is validly attached to a developer's personal GitHub account, Git operations against the organization's repositories will fail until the developer authenticates through their company's SSO portal.

---

## 13. Team Lead Perspective: Real Teams Mein Iska Use

1. **Automated Onboarding Diagnostic Script:**
   Junior developers onboarding ke pehle hafte mein authentication errors mein ghanto waste karte hain. Team Lead ek 10-line diagnostic script provide karta hai `diagnose_git.sh`:
   ```bash
   echo "Checking SSH Agent..." && ssh-add -l
   echo "Testing GitHub Port 22..." && ssh -T git@github.com -o ConnectTimeout=5 || ssh -T -p 443 git@ssh.github.com
   echo "Verifying Git Author..." && git config --get-regexp user
   ```
2. **Organization-Level Deploy Key Auditing:**
   Tech Lead periodic audit chalata hai GitHub GraphQL API use karke taaki dead CI deploy keys aur write-access tokens detect karke delete kiye ja sakein.

---

### 🔗 Official Documentation References:
- [GitHub Docs - Error: Permission Denied (publickey)](https://docs.github.com/en/authentication/troubleshooting-ssh/error-permission-denied-publickey)
- [GitHub Docs - GitHub's SSH Key Fingerprints](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/githubs-ssh-key-fingerprints)
- [GitHub Docs - Authorizing an SSH Key for Use with SAML Single Sign-On](https://docs.github.com/en/enterprise-cloud@latest/authentication/authenticating-with-saml-single-sign-on/authorizing-an-ssh-key-for-use-with-saml-single-sign-on)

