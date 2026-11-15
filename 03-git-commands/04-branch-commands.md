# Chapter 04: Branch Commands (`branch`, `switch`, `checkout` & Upstream Tracking)

---

## 1. Concept: Ye Kya Hai?

Git ka sabse powerful superpower hai **Cheap, Instant Branching**.

Is chapter mein hum branches ke management CLI suite ko master karenge:
- **`git branch`**: Branches ko list karna, create karna, rename karna, aur delete karna.
- **`git switch`**: Branches ke beech safely move karna (Modern Git 2.23+ standard).
- **`git checkout`**: The legacy overloaded swiss-army knife command, aur kyu industry ab `git switch` par shift ho chuki hai.
- **Upstream Tracking (`-vv`)**: Local branch ko remote branch ke sath synchronize aur compare karna.

---

## 2. Why: Iski Zarurat Kyu Padi? (The `git checkout` Design Flaw)

Git ke purane versions (pre-2.23) mein `git checkout` Git ki sabse confusing command maani jaati thi. 

Kyun? Kyunki ek hi command se:
1. Branch switch hoti thi: `git checkout feature`
2. Commit par Detached HEAD hota tha: `git checkout 4f8a192`
3. Working tree file discard hoti thi: `git checkout -- server.js` (Destructive action!)

Agar kisi developer se typo ho jata:
`git checkout main` ki jagah `git checkout main.js`
Toh Git branch switch karne ke bajaye **`main.js` ke uncommitted code ko permanently delete kar deta tha!**

### The Git 2.23 Architectural Split:
Git maintainers ne is confusion ko hamesha ke liye solve karne ke liye responsibilities ko do dedicated commands mein baant diya:
- **`git switch`**: Sirf aur sirf branches aur commits par navigate karne ke liye (`🟡 [CAUTION]`).
- **`git restore`**: Sirf aur sirf files ko stage/unstage ya discard karne ke liye (`🟡 [CAUTION]` / `🔴 [DANGEROUS]`).

---

## 3. Real-World Example: The "Dirty Working Tree" Checkout Block

Aap ek backend microservice par kaam kar rahe hain. 
Aapne `src/payment.js` mein 20 lines ka uncommitted code likha hua hai.
Achanak aapki team ke Slack par message aata hai: *"Lead ne `feature/auth` par ek change kiya hai, turant checkout karke review karo!"*

Aap terminal par chalate hain:
`git switch feature/auth`

Git achanak error throw karke ruk jata hai:
`error: Your local changes to the following files would be overwritten by checkout:`
`    src/payment.js`
`Please commit your changes or stash them before you switch branches.`
`Aborting`

**Why did Git stop?**
Kyunki `feature/auth` branch par `src/payment.js` ka code aapki current branch se differ kar raha tha! Agar Git switch allow kar deta, toh aapka uncommitted code **overwrite** ho jata! Git ne aapke 4 ghante ke kaam ko destroy hone se bacha liya.

---

## 4. Internal Working: What Happens When You Switch Branches?

Jab aap `git switch feature` chalate hain:

1. **Safety Check:** Git dekhta hai ki kya aapke working directory mein koi aisi modified file hai jo target branch ke commit mein bhi modified hai. Agar haan, toh abort karta hai.
2. **HEAD Reference Update:** Git `.git/HEAD` file ko open karke update karta hai:
   `ref: refs/heads/feature`
3. **Tree Reconstruction:** Git target commit ke Tree object ko read karta hai.
4. **Staging Index Update:** `.git/index` binary file ko target commit ke state par load karta hai.
5. **Disk File Replacement:** Operating system par files ko update, create, ya delete karta hai taaki disk target branch ke snapshot se 100% match kare.

```text
git switch feature Execution Flow:
┌──────────────┐      ┌───────────────┐      ┌───────────────┐      ┌───────────────┐
│ Safety Check │ ───► │ Update .git/  │ ───► │ Update Binary │ ───► │ Swap Files On │
│ (Dirty Files)│      │     HEAD      │      │  .git/index   │      │ Working Disk  │
└──────────────┘      └───────────────┘      └───────────────┘      └───────────────┘
```

---

## 5. Commands: Branch Management Suite

| Command | Safety | Description |
|---|:---:|---|
| `git branch` | `🟢 [SAFE]` | Local branches list karta hai. Active branch `*` se marked hoti hai. |
| `git branch -a` | `🟢 [SAFE]` | Local aur Remote-tracking (`remotes/origin/*`) dono branches list karta hai. |
| `git branch -vv` | `🟢 [SAFE]` | **Crucial:** Har branch ka upstream tracking aur sync status (`ahead 2, behind 1`) dikhata hai. |
| `git branch --merged` | `🟢 [SAFE]` | Wo saari branches dikhata hai jo current active branch mein already fully merge ho chuki hain. |
| `git branch --no-merged` | `🟢 [SAFE]` | Wo branches jo abhi tak merge nahi hui hain (Contains unmerged work!). |
| `git switch <branch>` | `🟡 [CAUTION]` | Existing branch par safely switch karta hai. |
| `git switch -c <name>` | `🟡 [CAUTION]` | Nayi branch create karke turant us par switch karta hai (Modern equivalent of `checkout -b`). |
| `git branch -m <old> <new>` | `🟡 [CAUTION]` | Local branch ka naam rename karta hai. |
| `git branch -d <branch>` | `🟡 [CAUTION]` | **Safe Delete:** Tabhi delete karega agar branch fully merged ho. Unmerged hone par Git reject karega! |
| `git branch -D <branch>` | `🔴 [DANGEROUS]` | **Force Delete:** Chahe work merge hua ho ya nahi, branch pointer permanently drop kar dega! |

---

## 6. Expected Output: Terminal Par Verification

```bash
# Detailed upstream tracking view:
$ git branch -vv
* main             7e9b2c3 [origin/main: ahead 1] feat(api): add health check
  feature/billing  4f8a192 [origin/feature/billing: behind 3] feat: stripe hook
  feature/legacy   1a2b3c4 [origin/feature/legacy: gone] chore: old sprint work

# Checking safe merged branches before cleanup:
$ git branch --merged
* main
  feature/login-fix   <-- Safe to delete with `git branch -d`!

# Attempting to delete an unmerged branch with safe flag -d:
$ git branch -d feature/billing
error: The branch 'feature/billing' is not fully merged.
If you are sure you want to delete it, run 'git branch -D feature/billing'.
# Git actively protects you from accidental data loss!
```

---

## 7. Visual Explanation: Upstream Tracking Status Breakdown

Jab aap `git branch -vv` chalate hain:

```text
Status Badge              Meaning
────────────              ───────
[origin/main]             Local branch and Remote branch are at the exact same commit.
[origin/main: ahead 2]    Aapne 2 commits local kiye hain jo abhi remote par push nahi huye.
[origin/main: behind 3]   Remote par 3 naye commits aa chuke hain jo local repo mein pull nahi huye.
[ahead 1, behind 2]       Diverged! Dono side alag-alag commits ho chuke hain (Merge/Rebase needed).
[origin/main: gone]       Remote branch GitHub par PR merge hone ke baad delete ho chuki hai!
```

---

## 8. Common Mistakes: Developers Kya Galti Karte Hain?

1. **Branching Off the Wrong Base Branch:**
   - Mistake: Developer `feature/cart` par kaam kar raha tha. Achanak use naya task mila `feature/profile`. Usne `feature/cart` ke upar hi `git switch -c feature/profile` chala diya!
   - Result: Profile branch ke andar Cart ka sara unmerged code inherit ho gaya. PR review mein 50 unrelated files aa gayi.
   - Fix: Nayi branch hamesha clean base (e.g. `main`) se kaatein: `git switch -c feature/profile main`.
2. **Blindly Using `-D` (Force Delete):**
   - Habitually `git branch -D` chalane se unmerged experimental feature branches permanently drop ho jaati hain. Always try `-d` first!
3. **Renaming Local Branch Without Updating Remote:**
   - Renaming `main` to `production` locally does NOT rename it on GitHub! Remote ko alag se push aur set-upstream karna padta hai.

---

## 9. Troubleshooting Guide: The 7-Step Diagnostic Framework

Agar uncommitted changes ki wajah se branch switch block ho jaye:

```text
1. OBSERVE:    Command: `git switch target-branch`
               Error: "error: Your local changes would be overwritten by checkout...
                       Please commit your changes or stash them."

2. INSPECT:    Modified files check karein:
               `git status -s`

3. UNDERSTAND: Aapka uncommitted code aur target branch ka code ek hi file
               ko touch kar rahe hain. Git data loss hone se rok raha hai.

4. PLAN:       Option A: Agar changes temporary hain, toh unhe stash mein rakhein.
               Option B: Agar changes complete hain, toh commit karein.
               Option C: Agar changes bekaar hain, toh discard karein.

5. EXECUTE:    Step 1 (Stash): git stash push -m "WIP before switching"
               Step 2 (Switch): git switch target-branch
               Step 3 (Return & Pop later): git switch original-branch && git stash pop

6. VERIFY:     `git status` run karein; target branch clean checked out ho jayegi.

7. PREVENT:    Branch switch karne se pehle hamesha `git status` check karne ki
               aadat daalein.
```

---

## 10. Practice Lab: Branch Navigation & Upstream Synchronization 💻

Chaliye Linux terminal par full branch lifecycle execute karte hain:

```bash
# Step 1: Create sandbox
mkdir -p /tmp/git-branch-lifecycle-lab && cd /tmp/git-branch-lifecycle-lab
git init -b main

# Step 2: Make initial commit
echo "Core Engine" > engine.js
git add engine.js && git commit -m "feat: initial commit on main"

# Step 3: Create and switch using modern git switch
git switch -c feature/search
echo "Search logic" > search.js
git add search.js && git commit -m "feat: implement search"

# Step 4: Verify current branch
git branch
# Output shows:
# * feature/search
#   main

# Step 5: Switch back to main and verify search.js vanishes from disk!
git switch main
ls
# Notice: search.js is GONE from disk! It only exists in the feature/search commit!

# Step 6: Test Safe Delete vs Force Delete
# Try to delete feature/search with -d:
git branch -d feature/search
# Git throws ERROR: "The branch 'feature/search' is not fully merged."
# Git protected your code!

# Step 7: Merge the feature into main
git merge feature/search

# Step 8: Now delete with safe -d flag:
git branch -d feature/search
# Output: "Deleted branch feature/search (was ...)"
# Success! Because it was fully merged, Git allowed deletion!

# Cleanup
cd ~ && rm -rf /tmp/git-branch-lifecycle-lab
```

---

## 11. Challenge: Test Your Understanding 🧠 (Scenario-Based Problems)

### 📌 Scenario Challenge 1: The "Branched Off the Wrong Branch" Emergency
> **Problem Statement:**
> Ek developer `feature/payment-v1` par kaam kar raha tha (jisme 10 unapproved commits the).
> Achanak use ek critical bug assign hua.
> Bina `main` par switch kiye, usne direct chala diya:
> `git switch -c bugfix/login-redirect`
> 
> Usne bugfix code likha aur 2 commits kar diye.
> Jab usne GitHub par PR open ki, toh PR mein **12 commits** (10 Payment ke + 2 Bugfix ke) dikhne lage!
> Lead ne bola: *"Is bugfix branch ko payment branch se alag karo aur seedha `main` ke upar re-base karo taaki sirf wahi 2 commits PR mein dikhein!"*
> 
> **Question:** `git rebase --onto` ka use karke is branch ko surgically `main` ke upar kaise transplant karenge?

<details>
<summary><b>💡 Click to Reveal Complete Solution & Walkthrough</b></summary>

#### Solution: Surgical Branch Transplant via `git rebase --onto`

Ye Git ki sabse powerful architectural commands mein se ek hai!
Syntax: `git rebase --onto <new-base> <old-base> <branch-to-move>`

#### Command Execution:
```bash
# Transplant bugfix/login-redirect onto main, cutting away feature/payment-v1:
git rebase --onto main feature/payment-v1 bugfix/login-redirect
```

#### What happens internally:
Git `feature/payment-v1` aur `bugfix/login-redirect` ke beech ke diff (yani sirf wo 2 bugfix commits) ko extract karega aur unhe seedha `main` branch ke latest commit ke upar replay kar dega!
Saare 10 unmerged payment commits gayab ho jayenge, aur PR clean ho jayegi!
</details>

---

### 📌 Scenario Challenge 2: Renaming a Branch Everywhere (Local & Remote)
> **Problem Statement:**
> Aapki team ne branch banayi thi `feature/autho`.
> Typo ki wajah se naam galat ho gaya, aur branch already GitHub remote par push ho chuki thi:
> `origin/feature/autho`
> 
> Team Lead ne bola: *"Is branch ka naam rename karke `feature/auth` karo—locally bhi aur GitHub remote par bhi, aur local branch ka upstream tracking bhi update karo!"*
> 
> **Question:** Is operation ko step-by-step execute karne ka exact terminal sequence kya hoga?

<details>
<summary><b>💡 Click to Reveal Complete Solution & Walkthrough</b></summary>

#### Step-by-Step Production Renaming Sequence:

```bash
# Step 1: Switch to the wrong branch locally
git switch feature/autho

# Step 2: Rename local branch
git branch -m feature/auth

# Step 3: Push the new branch to remote and set upstream tracking
git push -u origin feature/auth

# Step 4: Delete the old typo branch from remote GitHub!
git push origin --delete feature/autho

# Step 5: Verify tracking
git branch -vv
```
Local aur remote dono side branch flawlessly update ho gayi!
</details>

---

### 📌 Scenario Challenge 3: Mass Cleanup of Stale Merged Branches
> **Problem Statement:**
> Ek developer ke local laptop par pichle 1 saal se 80 purani local branches jama ho chuki hain jo already `main` branch mein merge ho chuki hain.
> Developer ek-ek karke `git branch -d` chalate-chalate thak gaya hai.
> 
> **Question:** Linux shell pipeline use karke un saari branches ko ek sath safely delete karne ki command kya hogi jo `main` mein fully merge ho chuki hain (aur current active `main` branch ko delete hone se skip karein)?

<details>
<summary><b>💡 Click to Reveal Complete Solution & Walkthrough</b></summary>

#### Solution: Safe Automation Pipeline

```bash
# Make sure you are on main branch first:
git switch main

# List merged branches, filter out main/master, and delete:
git branch --merged | grep -v '^\*' | grep -v 'main' | grep -v 'master' | xargs -r git branch -d
```

#### Why this is safe:
- `git branch --merged` sirf wahi branches return karta hai jo fully merged hain.
- Lowercase `-d` flag ensure karta hai ki agar galti se koi unmerged branch aayi bhi, toh Git use abort kar dega.
- 80 stale branches 1 second mein clean ho jaati hain!
</details>

---

## 12. Interview Questions (FAANG & Product Companies)

### Q1: Why did Git 2.23 introduce `git switch` and `git restore` to replace `git checkout`?
**Answer:** `git checkout` suffered from severe UX overloading and violated the Single Responsibility Principle. It handled branch navigation, detached commit inspection, and destructive file modification discarding within the same command, creating dangerous confusion where a typo could permanently delete uncommitted code. `git switch` was introduced exclusively for branch/commit navigation, while `git restore` was introduced exclusively for unstaging and discarding working tree modifications.

### Q2: What is the difference between `git branch -d` and `git branch -D`?
**Answer:**
- `git branch -d` (safe delete) performs a reachability check against the current branch's DAG. If the branch contains commits that have not been integrated into the current branch or its upstream tracking branch, Git refuses deletion to prevent accidental data loss.
- `git branch -D` (force delete, equivalent to `git branch --delete --force`) bypasses the reachability check and immediately unlinks the 41-byte ref file from `.git/refs/heads/`, dropping the pointer regardless of whether changes were merged.

### Q3: What is stored inside `.git/refs/remotes/origin/main` vs `.git/refs/heads/main`?
**Answer:**
- `.git/refs/heads/main` is your **Local Branch**. You can directly commit to it, amend it, and advance its pointer locally.
- `.git/refs/remotes/origin/main` is a **Remote-Tracking Branch**. It is a read-only local cached copy representing the state of the `main` branch on the remote server (`origin`) as of the last `git fetch` or `git pull`. You cannot commit directly to it; it only moves during remote network operations.

---

## 13. Team Lead Perspective: Real Teams Mein Iska Use

1. **Standardized Branch Naming Conventions:**
   Tech Leads enforce branch prefix policies in company developer guidelines:
   - `feat/<jira-id>-<short-description>` (e.g. `feat/AUTH-102-jwt-login`)
   - `fix/<jira-id>-<short-description>` (e.g. `fix/PAY-409-webhook-timeout`)
   - `chore/<description>` (e.g. `chore/upgrade-node-22`)
   - `hotfix/<description>` (e.g. `hotfix/cve-security-patch`)
   CI pipelines reject PR creation if the branch name fails the regex pattern.
2. **Automating Stale Branch Deletion on GitHub:**
   Team Lead GitHub repository settings mein **"Automatically delete head branches"** enable karta hai. Jaise hi PR merge hoti hai, GitHub remote feature branch ko auto-delete kar deta hai taaki remote repository clutter-free rahe.

---

### 🔗 Official Documentation References:
- [Git SCM - Git Branching: Basic Branching and Merging](https://git-scm.com/book/en/v2/Git-Branching-Basic-Branching-and-Merging)
- [Git SCM - git-switch Documentation](https://git-scm.com/docs/git-switch)
- [Git SCM - git-branch Documentation](https://git-scm.com/docs/git-branch)

