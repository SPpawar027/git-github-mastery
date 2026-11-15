# Chapter 01: Core Day-to-Day Commands (`init`, `clone`, `status`, `add`, `commit`)

---

## 1. Concept: Ye Kya Hai?

Ye Git ke wo 5 fundamental porcelain commands hain jinhe har software engineer har roz 50+ baar run karta hai:
1. **`git init`**: Ek naya, clean version-controlled repository initialize karna.
2. **`git clone`**: Remote server (GitHub/GitLab) par maujood repository ko uski poori history ke sath apni local machine par download karna.
3. **`git status`**: Working tree aur staging area ke health aur state ko inspect karna.
4. **`git add`**: Working directory ke changes ko Staging Area (Index) mein load karna.
5. **`git commit`**: Staged changes ka ek permanent, immutable snapshot repository DAG mein seal karna.

In commands ko upar-upar se chalana asan hai, lekin ek **Senior Engineer** inke internal mechanics, advanced flags (`--depth`, `--sparse`, `-p`, `--amend`), aur performance impact ko samajhta hai.

---

## 2. Why: Iski Zarurat Kyu Padi?

Jab ek developer code likhta hai:
- Wo ek hi din mein 20 files modify karta hai.
- Unmein se 5 files feature code hoti hain, 2 files tests hoti hain, aur 13 files temporary debug logs aur build artifacts hoti hain.

Agar aapke paas granular control na ho:
- Aap build artifacts repo mein push kar denge.
- Large teams ke 100GB repository ko clone karte waqt aapka internet exhaust ho jayega.
- Junior developers "wip", "fixed", "done" jaise meaningless commits se commit history ko destroy kar denge.

---

## 3. Real-World Example: 2-Hour CI Build Optimization via Shallow Clone

Aap ek FinTech platform ke Technical Lead hain. 
Aapne notice kiya ki company ki GitHub Actions CI pipeline har PR par 12 minute sirf `git clone` karne mein laga rahi hai! 
Kyunki project 8 saal purana tha, aur uski 5GB history thi jisme 45,000 commits the.

Aapne CI configuration mein ek single flag add kiya:
`git clone --depth=1 ...`

**Result:**
- Git ne purane 44,999 commits download karna band kar diya!
- Git ne sirf latest commit ka snapshot download kiya (5GB se घटकर 45MB ho gaya).
- Clone time **12 minute se gir kar 4 second** ho gaya!
- Company ka har mahine \$15,000 CI compute cost bacha.

---

## 4. Internal Working: Under the Hood of `git clone`

Jab aap `git clone https://github.com/org/repo.git` chalate hain, toh Git background mein 5 sequential steps execute karta hai:

1. **Target Folder Creation:** Local disk par ek naya folder banata hai.
2. **`git init`:** Us folder ke andar `.git/` directory structure initialize karta hai.
3. **Remote Configuration:** Remote server ka URL `origin` ke roop mein `.git/config` mein configure karta hai:
   `[remote "origin"] url = ... fetch = +refs/heads/*:refs/remotes/origin/*`
4. **`git fetch origin` (Smart Protocol Handshake):** Remote server se objects ka compressed `.pack` file download karta hai aur local `.git/objects/` mein unpack/index karta hai.
5. **`git checkout`:** Remote ke default branch (`HEAD` pointer jo GitHub par set tha, usually `main`) ke commit tree ko local disk par unpack karke Working Directory files create karta hai.

```text
git clone Execution Pipeline:
┌───────────┐      ┌─────────────┐      ┌────────────────┐      ┌──────────────┐
│ git init  │ ───► │ Remote Add  │ ───► │ Fetch Packfile │ ───► │ Checkout HEAD│
│ (Creates) │      │  (origin)   │      │ (Smart Proto)  │      │(Working Tree)│
└───────────┘      └─────────────┘      └────────────────┘      └──────────────┘
```

---

## 5. Commands: Daily Core Suite

| Command | Safety | Description |
|---|:---:|---|
| `git init -b main` | `🟡 [CAUTION]` | Nayi repository initialize karta hai with modern default branch `main`. |
| `git clone <url>` | `🟡 [CAUTION]` | Poori remote repository aur uski complete history clone karta hai. |
| `git clone --depth 1 <url>` | `🟡 [CAUTION]` | **Shallow Clone:** Sirf latest commit download karta hai (Ultra-fast). |
| `git status` | `🟢 [SAFE]` | Full human-readable state report. |
| `git status -s` | `🟢 [SAFE]` | Short format status (`M`, `A`, `??`, `D`). |
| `git add <file>` | `🟡 [CAUTION]` | Specific file ko stage karta hai. |
| `git add -A` (ya `git add .`)| `🟡 [CAUTION]` | Saari new, modified, aur deleted files ko stage karta hai. |
| `git add -u` (update only) | `🟡 [CAUTION]` | Sirf already tracked modified/deleted files stage karta hai (new untracked files skip karta hai!). |
| `git commit -m "feat: message"`| `🟡 [CAUTION]` | Staged snapshot ko immutable commit object mein convert karta hai. |
| `git commit --amend --no-edit`| `🟡 [CAUTION]` | Last commit mein naye staged changes merge karta hai bina message change kiye. |

---

## 6. Expected Output: Terminal Par Verification

```bash
# Rapid status check:
$ git status -s
 M src/auth.js
A  src/jwt.js
D  src/old-token.js
?? config/temp.json

# Making an atomic commit:
$ git commit -m "feat(auth): integrate jwt token validation"
[main 7a2b3c1] feat(auth): integrate jwt token validation
 3 files changed, 84 insertions(+), 12 deletions(-)
 create mode 100644 src/jwt.js
 delete mode 100644 src/old-token.js
```

---

## 7. Visual Explanation: The 4 File Lifecycles in Git

```text
               ┌───────────────┐
               │   UNTRACKED   │ (New file created on disk)
               └───────┬───────┘
                       │ git add
                       ▼
               ┌───────────────┐
               │    STAGED     │ (In .git/index, ready for commit)
               └───────┬───────┘
                       │ git commit
                       ▼
 ┌───────────► ┌───────────────┐
 │             │  UNMODIFIED   │ (Committed & in sync with repository)
 │             └───────┬───────┘
 │ git commit          │ File edited
 │                     ▼
 │             ┌───────────────┐
 └──────────── │   MODIFIED    │ (Changes in working directory)
   (via staging)└───────────────┘
```

---

## 8. Common Mistakes: Developers Kya Galti Karte Hain?

1. **`git add .` vs `git add -A` vs `git add -u` Confusion:**
   - `git add .`: Current folder aur uske subfolders ke changes stage karta hai (parent folders ke changes chhod deta hai agar subfolder mein hon).
   - `git add -A`: Pure repository root se saare changes stage karta hai (new, modified, deleted).
   - `git add -u`: Sirf already tracked files ke modification/deletion stage karta hai. New untracked files ko touch nahi karta!
2. **Lazy Commit Messages:**
   - Antipattern: `git commit -m "update"` ya `git commit -m "fixed stuff"`.
   - Production Rule: Hamesha Conventional Commits format use karein: `<type>(<scope>): <short description>`.
3. **Committing Secrets via Blind `git add .`:**
   - Har commit se pehle hamesha `git status` aur `git diff --staged` run karna Senior Engineer ka basic discipline hai.

---

## 9. Troubleshooting Guide: The 7-Step Diagnostic Framework

Agar `git clone` slow network par fail ho jaye:

```text
1. OBSERVE:    Command: `git clone https://github.com/large-org/massive-repo.git`
               Error: "error: RPC failed; curl 56 GnuTLS recv error (-9): A TLS packet with unexpected length was received."
                      "fatal: early EOF"
                      "fatal: fetch-pack: invalid index-pack output"

2. INSPECT:    Repository size aur network buffer check karein:
               `git config --get http.postBuffer`

3. UNDERSTAND: Large packfile download karte waqt slow connection ya Wi-Fi glitch
               ki wajah se HTTP POST buffer overflow ho gaya aur TCP packet drop ho gaya.

4. PLAN:       Buffer size badhayein aur pure 10-year history ke bajaye
               shallow clone karke aage badhein.

5. EXECUTE:    Step 1: git config --global http.postBuffer 524288000
               Step 2: git clone --depth 1 https://github.com/large-org/massive-repo.git
               Step 3 (Optional history fetch): cd massive-repo && git fetch --unshallow

6. VERIFY:     `git status` run karein; repository successfully checked out honi chahiye.

7. PREVENT:    Unreliable internet par hamesha `--depth 1` shallow clone use karein.
```

---

## 10. Practice Lab: Mastering Core Porcelain Operations 💻

Follow this comprehensive hands-on terminal lab:

```bash
# Step 1: Create lab sandbox
mkdir -p /tmp/git-core-suite-lab && cd /tmp/git-core-suite-lab
git init -b main

# Step 2: Test git add -u vs git add .
echo "Initial Core" > core.js
git add core.js
git commit -m "feat: initial core module"

# Modify tracked file AND create an untracked file
echo "// Updated logic" >> core.js
echo "Temporary debug script" > debug.js

# Check status:
git status -s
# Output shows:
#  M core.js
# ?? debug.js

# Step 3: Use git add -u (Update tracked only)
git add -u
git status -s
# Notice: core.js is STAGED (M ), but debug.js remains UNTRACKED (??)!
# This is a lifesaver when you have messy scratch files!

# Commit the tracked change:
git commit -m "refactor(core): update execution logic"

# Step 4: The Clean Amend Operation (Local fix before pushing)
# Developer forgot to add author documentation in core.js:
echo "/** Author: Senior Lead */" >> core.js
git add core.js

# Merge this into previous commit without opening text editor:
git commit --amend --no-edit

# Verify commit log:
git log -1 --stat
# Notice: Only 1 clean commit exists in history with both updates!

# Step 5: Test Shallow Clone on a Real Remote (Public Repo)
cd /tmp
git clone --depth 1 https://github.com/git/git.git git-shallow-demo
cd git-shallow-demo

# Check how many commits were downloaded:
git rev-list --count HEAD
# Output: Exactly 1 commit! (Instead of 70,000+ commits!)
# Compare how fast this clone was!

# Cleanup
cd ~ && rm -rf /tmp/git-core-suite-lab /tmp/git-shallow-demo
```

---

## 11. Challenge: Test Your Understanding 🧠 (Scenario-Based Problems)

### 📌 Scenario Challenge 1: The Shallow Clone PR Push Dilemma
> **Problem Statement:**
> Ek developer ne fast setup ke liye shallow clone use kiya:
> `git clone --depth 1 git@github.com:company/api.git`
> 
> Usne feature branch banayi, 2 commits kiye, aur push kiya.
> Lekin jab usne GitHub par Pull Request open ki, toh PR bolti hai:
> *"Cannot merge: branches have diverged / unable to find common ancestor!"*
> Aur terminal par `git rebase main` fail ho gaya error dekar:
> `fatal: no common commits`
> 
> **Questions:**
> 1. Shallow clone mein rebasing aur common ancestor calculate karna fail kyu hua?
> 2. Existing shallow clone repository ko bina dobara clone kiye full repository (complete history) mein kaise convert karenge?

<details>
<summary><b>💡 Click to Reveal Complete Solution & Walkthrough</b></summary>

#### Solution & Forensic Explanation:

**1. Why it failed:**
Kyunki `--depth 1` clone ke paas sirf **latest commit** tha; uska koi parent ya grandparent history mein nahi tha! Jab developer ne rebase ya PR merge try kiya, Git ko common merge base dhundhne ke liye purani history chahiye thi jo shallow repo mein physically missing thi!

#### 2. The Conversion Command (`--unshallow`):
Developer ko repo delete karne ki zarurat nahi hai! Wo Git ko bol sakta hai ki missing purani history remote se fetch kar le:

```bash
git fetch --unshallow origin
```

#### What happens internally:
Git remote server se remaining 40,000 commits ke packfiles download karega aur shallow boundaries (`.git/shallow` file) ko delete kar dega. Repository full normal repo ban jayegi aur rebase/merge instantly work karega!
</details>

---

### 📌 Scenario Challenge 2: The Staging Scope Trap
> **Problem Statement:**
> Aapka project directory structure aisa hai:
> ```text
> my-app/
> ├── backend/
> │   └── server.js (Modified)
> └── frontend/
>     └── App.js (Modified)
> ```
> Developer `my-app/frontend/` folder ke andar terminal mein khada tha aur usne chalaya:
> `git add .`
> `git commit -m "update both frontend and backend"`
> 
> Lekin commit hone ke baad jab usne `git status` dekha, toh `backend/server.js` abhi bhi modified uncommitted show ho rahi thi!
> 
> **Question:** `git add .` ne backend ke changes ko stage kyu nahi kiya, aur current directory se pure repo root ko stage karne ki universal command kya hai?

<details>
<summary><b>💡 Click to Reveal Complete Solution & Walkthrough</b></summary>

#### Solution & Scope Mechanics:

**Why backend was missed:**
Dot (`.`) terminal shell mein **Current Working Directory** ko represent karta hai!
Kyunki developer `frontend/` folder ke andar tha, `git add .` ne sirf `frontend/` folder aur uske child files ko scan kiya. Parent aur sibling folders (`backend/`) ko Git ne completely ignore kar diya.

#### Universal Commands to Stage Whole Repo Regardless of Current Directory:
```bash
# Method 1: git add -A (All across entire repo root)
git add -A

# Method 2: Pathspec from repo root (Using colon-slash)
git add :/
```

`:/` pathspec Git ko explicitly command deta hai: *"Repo ke top-level root par jao aur wahan se saare changes stage karo, chahe main kisi bhi deep subfolder mein baitha hoon!"*
</details>

---

### 📌 Scenario Challenge 3: Atomic Staging with Untracked Files Bypass
> **Problem Statement:**
> Aapke working directory mein 10 modified files hain aur 20 nayi generated untracked files (`test-report-1.html`, `dump.json`, etc.) hain jo abhi `.gitignore` mein add nahi hui hain.
> Aapko sirf un 10 already tracked modified files ko stage karke commit karna hai bina kisi bhi nayi untracked file ko stage kiye.
> 
> **Question:** Kaunsi single flag wali Git command bina manual file names type kiye ye execute karegi?

<details>
<summary><b>💡 Click to Reveal Complete Solution & Walkthrough</b></summary>

#### Solution: `git add -u` (Update Flag)

```bash
git add -u
git commit -m "fix(core): apply fixes to tracked modules"
```

#### Why this works:
`-u` (short for `--update`) Git ko instruct karta hai:
- Sirf un files ke changes stage karo jo already Git ke Staging Index mein tracked hain.
- Saari new (untracked `??`) files ko safely ignore kar do!
</details>

---

## 12. Interview Questions (FAANG & Product Companies)

### Q1: What is a Shallow Clone (`git clone --depth N`) and what are its trade-offs?
**Answer:** A shallow clone creates a truncated commit history containing only the latest $N$ commits. 
- **Benefits:** Massive bandwidth savings, 95% faster download times, minimal disk footprint (ideal for CI/CD runners and deployment containers).
- **Trade-offs:** You cannot perform deep forensic debugging (`git blame` or `git log` beyond $N$ commits will fail), and branching/merging operations that require traversing back to older common ancestors may fail unless unshallowed (`git fetch --unshallow`).

### Q2: What is the exact difference between `git add .`, `git add -A`, and `git add -u`?
**Answer:**
- `git add .`: Stages new, modified, and deleted files strictly within the current working directory and its subdirectories.
- `git add -A` (or `--all`): Stages all new, modified, and deleted files across the **entire repository**, regardless of your current terminal subdirectory.
- `git add -u` (or `--update`): Stages only modifications and deletions of **already tracked** files across the repository. It completely ignores new untracked files.

### Q3: What is the internal difference between `git init` and `git init --bare`?
**Answer:**
- `git init`: Creates a working directory with a hidden `.git` folder containing objects and refs. Used by developers on workstations to write and test code.
- `git init --bare`: Creates a repository without a working directory (no files checked out to disk). All repository internal databases reside directly in the root folder. Used exclusively on central remote servers (GitHub, GitLab, central SSH servers) where direct editing never occurs.

---

## 13. Team Lead Perspective: Real Teams Mein Iska Use

1. **Enforcing Conventional Commits via Commitlint:**
   Junior developers commit messages like "fix", "done", "test". Team Lead project mein **commitlint** aur **Husky `commit-msg` hook** enforce karta hai:
   Commit message format mandatory hota hai: `type(scope): subject` (e.g. `feat(auth): add google sso login`). Agar developer format follow nahi karta, toh terminal commit ko reject kar deta hai.
2. **CI/CD Shallow Clone Governance:**
   Lead CI infrastructure mein mandate karta hai ki testing aur build jobs hamesha `actions/checkout` with `fetch-depth: 1` run karein taaki deployment speed maximum rahe, jabki sonar/code-coverage jobs with `fetch-depth: 0` run karein jahan deep blame analysis zaroori hoti hai.

---

### 🔗 Official Documentation References:
- [Git SCM - git-init Documentation](https://git-scm.com/docs/git-init)
- [Git SCM - git-clone Documentation](https://git-scm.com/docs/git-clone)
- [Git SCM - git-add Documentation](https://git-scm.com/docs/git-add)
- [Conventional Commits Specification](https://www.conventionalcommits.org/en/v1.0.0/)

