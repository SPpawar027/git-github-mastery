# Chapter 04: Multiple GitHub Accounts on One Machine (The Ultimate SSH Host Routing Guide)

---

## 1. Concept: Ye Kya Hai?

Real-world development mein 90% software engineers ke paas **do alag-alag GitHub accounts** hote hain:
1. **Account 1 (Work / Corporate):** Company organization repositories access karne ke liye (e.g. `alice-work`).
2. **Account 2 (Personal / Open-Source):** Personal side projects aur open-source contributions ke liye (e.g. `alice-personal`).

### The Big Problem:
By default, jab aap `git push origin main` chalate hain, Git connect karta hai `git@github.com` par.
Aapka computer ek time par ek hi default SSH key bhejta hai.
- Agar aapne work key bheji, toh personal repository bolegi: `Permission denied to alice-work`.
- Agar aapne personal key bheji, toh company repository bolegi: `Permission denied to alice-personal`.

Is chapter mein hum industry-standard **SSH Host Aliasing (`~/.ssh/config`)** aur **Git Conditional Includes (`includeIf`)** ko setup karenge taaki aapka computer repository ke folder aur URL ke basis par automatically correct key aur correct author email pick kar le!

---

## 2. Why: Iski Zarurat Kyu Padi? (The Account Collision Friction)

Agar aap ye proper setup na karein:
- Aapko har roz SSH keys rename karni padengi ya `ssh-add -D` karke bar-bar keys switch karni padengi.
- Galti se company ka confidential intellectual property (code) aapke personal GitHub account par push ho jayega (Legal / Compliance violation!).
- Personal open-source commits mein company ki corporate email chali jayegi.

### The Clean Architecture Solution:
Hum apne system ko do layers par organize karenge:
1. **Network Authentication Layer (`~/.ssh/config`):** Decides which **SSH Private Key** to use for GitHub authentication.
2. **Commit Metadata Layer (`~/.gitconfig` with `includeIf`):** Decides which **Author Name & Email** to embed inside the commit object.

---

## 3. Real-World Example: Contractor Working for Two Clients

Aap ek Senior Consultant hain:
- Subah aap Client A (`client-alpha`) ke backend repo par kaam karte hain.
- Shaam ko aap Client B (`client-beta`) ke frontend repo par kaam karte hain.
- Raat ko aap apne personal GitHub portfolio par commit karte hain.

Bina kisi manual logout ke, bina kisi password prompt ke, aur bina kisi configuration file ko touch kiye:
- Client A repo mein push karte hi Client A ki SSH key aur corporate email lagti hai.
- Client B repo mein Client B ki key lagti hai.
- Personal repo mein personal key lagti hai.
Ye 100% automated isolation hai!

---

## 4. Internal Working: SSH Host Aliasing & `IdentitiesOnly`

Jab aap SSH command chalate hain:
`git clone git@github.com:org/repo.git`
SSH client dekhta hai hostname hai `github.com`.

Lekin hum OpenSSH ko **Custom Virtual Hostnames (Aliases)** de sakte hain:
- `github-work`
- `github-personal`

Dono aliases internet par exact wahi physical server `github.com` ko point karte hain, lekin alag-alag private keys load karte hain!

```text
~/.ssh/config Configuration Architecture:

Host github-work
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_work
    IdentitiesOnly yes

Host github-personal
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_personal
    IdentitiesOnly yes
```

### The Super-Critical `IdentitiesOnly yes` Flag:
Ye flag kyun mandatory hai?
Default taur par OpenSSH agent memory mein loaded saari keys ko ek-ek karke try karta hai (Key 1 try ki, Key 2 try ki).
GitHub 3-5 failed key attempts ke baad connection reject kar deta hai:
`Received disconnect from ...: 2: Too many authentication failures`.
`IdentitiesOnly yes` OpenSSH ko strictly bolta hai: *"Sirf aur sirf wahi specific IdentityFile use karo jo is Host block ke andar explicitly define ki gayi hai! Baaki keys ko touch mat karo!"*

---

## 5. Commands: Multi-Account Generation & Testing Suite

| Command | Safety | Description |
|---|:---:|---|
| `ssh-keygen -t ed25519 -C "work@corp.com" -f ~/.ssh/id_ed25519_work` | `🟡 [CAUTION]` | Generates dedicated Work SSH key pair. |
| `ssh-keygen -t ed25519 -C "pers@gmail.com" -f ~/.ssh/id_ed25519_personal` | `🟡 [CAUTION]` | Generates dedicated Personal SSH key pair. |
| `ssh -T git@github-work` | `🟢 [SAFE]` | **Test Account 1:** Verifies connection using the work alias. |
| `ssh -T git@github-personal` | `🟢 [SAFE]` | **Test Account 2:** Verifies connection using the personal alias. |
| `git clone git@github-work:org/repo.git` | `🟡 [CAUTION]` | Clones repo using Account 1 authentication. |
| `git clone git@github-personal:user/repo.git` | `🟡 [CAUTION]` | Clones repo using Account 2 authentication. |

---

## 6. Expected Output: Terminal Par Live Verification

Jab aap dono host aliases test karenge, terminal clearly do alag usernames recognize karega:

```bash
# Testing Account 1 (Work):
$ ssh -T git@github-work
Hi alice-corporate! You've successfully authenticated, but GitHub does not provide shell access.

# Testing Account 2 (Personal):
$ ssh -T git@github-personal
Hi alice-personal-coder! You've successfully authenticated, but GitHub does not provide shell access.
```

Notice the miracle: **Ek hi computer par dono accounts simultaneous live authenticate ho rahe hain!**

---

## 7. Visual Explanation: Traffic Routing via `~/.ssh/config`

```text
                     ┌────────────────────────────────────────────────────────┐
                     │            DEVELOPER LAPTOP TERMINAL                   │
                     └──────────────────────────┬─────────────────────────────┘
                                                │
                 ┌──────────────────────────────┴──────────────────────────────┐
                 ▼                                                             ▼
     git push to github-work                                       git push to github-personal
                 │                                                             │
                 ▼                                                             ▼
┌─────────────────────────────────┐                           ┌─────────────────────────────────┐
│ ~/.ssh/config                   │                           │ ~/.ssh/config                   │
│ Host github-work                │                           │ Host github-personal            │
│ Target: github.com              │                           │ Target: github.com              │
│ Key: ~/.ssh/id_ed25519_work     │                           │ Key: ~/.ssh/id_ed25519_personal │
└────────────────┬────────────────┘                           └────────────────┬────────────────┘
                 │                                                             │
                 │ Authenticates as @alice-corporate                           │ Authenticates as @alice-personal
                 ▼                                                             ▼
┌───────────────────────────────────────────────────────────────────────────────────────────────┐
│                                GITHUB CLOUD SERVERS (github.com)                              │
│                                                                                               │
│  ├── Company Organization Repos (Write granted to @alice-corporate)                           │
│  └── Personal Repositories      (Write granted to @alice-personal)                            │
└───────────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 8. Common Mistakes: Developers Kya Galti Karte Hain?

1. **Cloning with Standard `github.com` URL:**
   - Mistake: Account 2 ke repo ko `git clone git@github.com:...` se clone kar lena.
   - Result: Git default work key bhej dega aur permission denied error aayega!
   - Rule: Account 2 ke liye URL mein `github.com` ko replace karke custom alias likhein: `git clone git@github-personal:...`!
2. **Forgetting `IdentitiesOnly yes`:**
   - Result: `Too many authentication failures` error.
3. **Mismatched Git Author Email:**
   - Push toh Account 2 se ho gaya, lekin local commit ke andar work email chali gayi kyunki global config work par set thi. (Fix: Use `includeIf:gitdir:`).

---

## 9. Troubleshooting Guide: The 7-Step Diagnostic Framework

Agar Account 2 ke repo mein push karte waqt Account 1 permission error aaye:

```text
1. OBSERVE:    Command: `git push origin main`
               Error: "ERROR: Permission to personal-user/blog.git denied to work-user."
                      "fatal: Could not read from remote repository."

2. INSPECT:    Remote URL check karein:
               `git remote -v`
               Notice: `origin git@github.com:personal-user/blog.git`
               It has standard "github.com", NOT our alias!

3. UNDERSTAND: Git default SSH host use kar raha hai, jisse system default work key
               bhej raha hai. Remote repo ko personal alias par point karna hoga.

4. PLAN:       Remote URL ke hostname `github.com` ko badal kar `github-personal` karein.

5. EXECUTE:    git remote set-url origin git@github-personal:personal-user/blog.git

6. VERIFY:     `git remote -v` verify karein aur `git push origin main` run karein;
               push instantly succeed ho jayega!

7. PREVENT:    Repo clone karte waqt URL ko modify karke paste karein:
               `git clone git@github-personal:username/repo.git`.
```

---

## 10. Practice Lab: Complete End-to-End Dual-Account Setup 💻

Is practical lab mein hum apne terminal par complete dual-account infrastructure create karenge:

```bash
# Step 1: Create dedicated SSH directory if not exists
mkdir -p ~/.ssh
chmod 700 ~/.ssh

# Step 2: Generate Key Pair for Account 1 (Simulated Work)
ssh-keygen -t ed25519 -C "dev-account1@company.com" -f ~/.ssh/id_ed25519_acc1 -N ""

# Step 3: Generate Key Pair for Account 2 (Simulated Personal)
ssh-keygen -t ed25519 -C "dev-account2@gmail.com" -f ~/.ssh/id_ed25519_acc2 -N ""

# Step 4: Set strict permissions
chmod 600 ~/.ssh/id_ed25519_acc1 ~/.ssh/id_ed25519_acc2
chmod 644 ~/.ssh/id_ed25519_acc1.pub ~/.ssh/id_ed25519_acc2.pub

# Step 5: Configure ~/.ssh/config with Host Aliasing!
cat << 'EOF' >> ~/.ssh/config

# --- Account 1 (Work / Primary) ---
Host github-work
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_acc1
    IdentitiesOnly yes

# --- Account 2 (Personal / Secondary) ---
Host github-personal
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_acc2
    IdentitiesOnly yes
EOF

chmod 600 ~/.ssh/config

# Step 6: Setup Directory-Based Automatic Git Config (includeIf)
# Create directories:
mkdir -p ~/git-work ~/git-personal

# Create work git config:
cat << 'EOF' > ~/.gitconfig-work
[user]
    name = Dev Account 1
    email = dev-account1@company.com
EOF

# Create personal git config:
cat << 'EOF' > ~/.gitconfig-personal
[user]
    name = Dev Account 2
    email = dev-account2@gmail.com
EOF

# Add conditional routing into main ~/.gitconfig:
cat << 'EOF' >> ~/.gitconfig

[includeIf "gitdir:~/git-work/"]
    path = ~/.gitconfig-work

[includeIf "gitdir:~/git-personal/"]
    path = ~/.gitconfig-personal
EOF

# Step 7: VERIFY AUTOMATIC CONTEXT SWITCHING!
cd ~/git-work && mkdir -p test-work && cd test-work && git init
git config user.email
# Output: dev-account1@company.com (AUTOMATICALLY!)

cd ~/git-personal && mkdir -p test-personal && cd test-personal && git init
git config user.email
# Output: dev-account2@gmail.com (AUTOMATICALLY!)

# Cleanup temporary test folders
rm -rf ~/git-work/test-work ~/git-personal/test-personal
```

---

## 11. Challenge: Test Your Understanding 🧠 (Scenario-Based Problems)

### 📌 Scenario Challenge 1: The SSH URL Conversion Formula
> **Problem Statement:**
> Aapne GitHub web UI se personal project ka SSH clone link copy kiya:
> `git@github.com:my-personal-user/react-portfolio.git`
> 
> Aapke laptop par `~/.ssh/config` mein personal alias configured hai: `Host github-personal`.
> 
> **Question:** 
> 1. Agar aap is exact URL ko clone karenge, toh kya hoga?
> 2. Is URL ko custom alias support karne ke liye kaise modify karenge?

<details>
<summary><b>💡 Click to Reveal Complete Solution & Walkthrough</b></summary>

#### Solution:

**1. What happens if cloned directly:**
OpenSSH URL ke `github.com` part ko dekhega aur aapki default primary work key bhej dega. GitHub bolega:
`ERROR: Repository not found` ya `Permission denied to work-user`.

#### 2. The Conversion Formula:
URL mein se `github.com` ko delete karke apna configured alias name paste karein:
- **Original URL:** `git@github.com:my-personal-user/react-portfolio.git`
- **Aliased URL:** `git@github-personal:my-personal-user/react-portfolio.git`

#### Clone Command:
```bash
git clone git@github-personal:my-personal-user/react-portfolio.git
```
Git ab automatically `~/.ssh/id_ed25519_acc2` use karega aur clone flawlessly execute ho jayega!
</details>

---

### 📌 Scenario Challenge 2: Retrofitting an Already Cloned Repository
> **Problem Statement:**
> Ek developer ne 6 mahine pehle ek client repo clone kiya tha.
> Repo already cloned hai aur uske andar 15 local feature branches hain.
> Developer ne naya SSH multi-account config banaya jisme client ka alias hai: `github-client`.
> 
> Developer soch raha hai: *"Kya mujhe repo ko dobara clone karna padega?"*
> 
> **Question:** Bina repo dobara clone kiye, existing local repo ke remote origin ko naye alias par update karne ki single command kya hai?

<details>
<summary><b>💡 Click to Reveal Complete Solution & Walkthrough</b></summary>

#### Solution: `git remote set-url`

Aapko repo delete karne ya dobara clone karne ki **0% zarurat hai**!
Sirf existing remote origin ka URL update kar dein:

```bash
git remote set-url origin git@github-client:client-org/core-backend.git
```

#### Verification:
```bash
git remote -v
git fetch origin
```
Saari 15 local branches intact hain aur push/pull naye aliased SSH key se chalne laga!
</details>

---

## 12. Interview Questions (FAANG & Product Companies)

### Q1: How do you configure a single machine to interact with two different GitHub accounts via SSH?
**Answer:** The industry standard solution uses a two-part approach:
1. **SSH Layer (`~/.ssh/config`):** Generate two separate SSH key pairs (e.g. `id_ed25519_work` and `id_ed25519_personal`) and define two distinct Host aliases (`Host github-work` and `Host github-personal`) pointing to `HostName github.com` with `IdentitiesOnly yes`. Repositories are cloned using the aliased hostname (e.g. `git@github-personal:user/repo.git`).
2. **Git Author Layer (`~/.gitconfig`):** Use directory-based conditional includes (`[includeIf "gitdir:~/work/"]`) to automatically inject the appropriate corporate vs personal `user.name` and `user.email`.

### Q2: Why is the `IdentitiesOnly yes` directive mandatory in `~/.ssh/config` when managing multiple accounts?
**Answer:** By default, OpenSSH offers all keys currently loaded in the SSH Agent in sequential order until one succeeds. If GitHub attempts to authenticate against the wrong key multiple times, it triggers an authentication failure or locks the connection with `Too many authentication failures`. `IdentitiesOnly yes` forces OpenSSH to exclusively present the key specified in the matching `IdentityFile` directive, preventing key crosstalk across accounts.

### Q3: What happens if you push a commit with Author Email A using SSH Key B?
**Answer:** The push succeeds (provided Account B has write permissions to the repository). On GitHub's UI, the commit author avatar will link to whichever profile registered Email A, but the repository audit log, push webhook, and security events will record that the actual network operation was authenticated and executed by Account B.

---

## 13. Team Lead Perspective: Real Teams Mein Iska Use

1. **BYOD (Bring Your Own Device) Security Compliance:**
   Jab engineers personal laptops par remote corporate work karte hain, Tech Lead security checklist mandate karta hai:
   - Corporate repositories must be isolated inside `~/work/` with strict `includeIf` policies.
   - Corporate SSH keys must be password-protected with 16+ character passphrases.
   - Code from company repositories must never be pushed through personal GitHub account credentials.
2. **Pre-Push Hook Domain Audit:**
   Team Lead enterprise repositories mein local pre-push hook distribute karta hai jo check karta hai:
   ```bash
   if [[ $(git config user.email) != *"@mycompany.com" ]]; then
       echo "FATAL: Attempting to push with non-corporate email! Aborting."
       exit 1
   fi
   ```

---

### 🔗 Official Documentation References:
- [OpenSSH Config Manual](https://man.openbsd.org/ssh_config)
- [Git SCM - Conditional Includes (`includeIf`)](https://git-scm.com/docs/git-config#_includes)
- [GitHub Docs - Managing Multiple Accounts](https://docs.github.com/en/account-and-profile/setting-up-and-managing-your-personal-account-on-github/managing-your-personal-account/managing-multiple-accounts)

