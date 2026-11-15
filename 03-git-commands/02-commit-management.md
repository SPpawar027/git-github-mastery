# Chapter 02: Commit Management (Atomic Commits, Amending & Author Metadata)

---

## 1. Concept: Ye Kya Hai?

Ek amateur developer aur ek **Senior Technical Lead** ke beech ka sabse bada farq unke commits ki quality se dikhta hai.

**Atomic Commit** ka matlab hota hai:
Ek aisa commit jo sirf aur sirf **ek single logical task** ya feature ko encapsulate karta ho.
- Agar wo commit revert kiya jaye, toh sirf wahi feature/bugfix revert hoga, baaki pura project seamlessly chalta rahega.
- Us commit ke saare unit tests 100% pass honge (no broken intermediate commits).

Is chapter mein hum commit crafting ki art, amending mechanics, author vs committer metadata, aur GPG cryptographic signing ko deep dive karenge.

---

## 2. Why: Iski Zarurat Kyu Padi? (The `git bisect` Superpower)

Sochiye aapke production server par achanak ek mysterious bug report hua:
*"Cart checkout par discount coupon apply nahi ho raha!"*

Pichle 2 hafton mein team ne 200 commits kiye hain.
- **Scenario A (Messy Commits):**
  Developers ne giant commits kiye the: *"misc fixes and updates"* jisme ek hi commit mein CSS, database query, aur discount coupon sab mix tha.
  Debugging karna nightmare ban jata hai. Aap single line revert nahi kar sakte kyunki pura frontend toot jayega.

- **Scenario B (Atomic Commits):**
  Har task ka alag commit tha.
  Aap automated debugging tool **`git bisect`** chalate hain.
  Git binary search se 200 commits ko sirf 7 steps mein test karta hai aur exact 4-line commit pakad leta hai:
  `fix(coupon): handle case-insensitive discount codes`
  Lead use 1 second mein inspect karta hai aur revert kar deta hai. Incident resolved!

---

## 3. Real-World Example: Author vs Committer in Open Source & Cherry-Picks

Aapne kabhi notice kiya hai ki Git har commit ke andar **do alag identities** store karta hai?
1. **Author:** Jis insaan ne original code likha.
2. **Committer:** Jis insaan ne us code ko repository history mein physically inject ya apply kiya.

### Real Scenario:
- **Developer Alice** ne ek bugfix likha aur patch email kiya ya feature branch par commit kiya.
- **Lead Bob** ne Alice ke code ko review kiya aur use main branch par `git cherry-pick` ya rebase karke commit kiya.
- Git dono ka credit preserve karta hai:
  - `Author: Alice <alice@work.com>` (Alice gets the engineering credit)
  - `Committer: Bob <bob@work.com>` (Bob gets the audit accountability for merging it)

---

## 4. Internal Working: What Actually Happens During `git commit --amend`?

Bohot se developers sochte hain ki `git commit --amend` existing commit object ke andar jakar text ko "edit" kar deta hai.

**Fact:** Git objects completely immutable hote hain! Git kisi purane commit ko modify kar hi nahi sakta!

### The Amend Lifecycle:
1. Git Staging Area ka naya Tree object banata hai.
2. Git ek **bilkul naya Commit Object** generate karta hai naye SHA hash ke sath!
3. Is naye commit ka parent wahi grandparent hota hai jo purane commit ka parent tha.
4. Active branch pointer (`refs/heads/main`) ko move karke naye commit par point kar deta hai.
5. **Purana commit object delete nahi hota!** Wo `.git/objects/` mein **dangling commit** ban jata hai, jo `git reflog` mein zinda rehta hai.

```text
BEFORE AMEND:
Commit A (Grandparent) ───► Commit B (SHA: 1111) ◄─── main (HEAD)


AFTER RUNNING git commit --amend:
Commit A (Grandparent) ───► Commit C (SHA: 9999) ◄─── main (HEAD)
        │
        └─────────────────► [Commit B (SHA: 1111)] (Orphaned / Hidden)
```

> [!WARNING]
> Agar aapne Commit B ko already GitHub par push kar diya tha, aur fir local machine par `--amend` karke Commit C bana diya, toh remote server aur local repo diverge ho jayenge! Agla `git push` reject ho jayega (`non-fast-forward rejection`). Isiliye: **Never amend commits that have already been pushed to a shared public branch!**

---

## 5. Commands: Commit Crafting Suite

| Command | Safety | Description |
|---|:---:|---|
| `git commit -m "feat: message"` | `🟡 [CAUTION]` | Clean atomic commit banata hai. |
| `git commit --amend` | `🟡 [CAUTION]` | Last commit ka message update karne ke liye CLI editor open karta hai. |
| `git commit --amend --no-edit` | `🟡 [CAUTION]` | Staged changes ko last commit mein silently merge karta hai bina message change kiye. |
| `git commit --amend --reset-author` | `🟡 [CAUTION]` | Current updated git config se Author identity re-calculate karta hai. |
| `git commit -S -m "feat: message"` | `🟡 [CAUTION]` | Commit ko GPG cryptographic key se digitally sign karta hai. |
| `git log --show-signature -1` | `🟢 [SAFE]` | Commit ka GPG cryptographic signature aur verification badge inspect karta hai. |

---

## 6. Expected Output: Terminal Par Verification

```bash
# Making an initial commit:
$ git commit -m "feat(api): add health check endpoint"
[main 4f8a192] feat(api): add health check endpoint
 1 file changed, 12 insertions(+)

# Realizing you forgot to stage the test file, adding it, and amending:
$ git add tests/health.test.js
$ git commit --amend --no-edit
[main 7e9b2c3] feat(api): add health check endpoint
 Date: Sat Nov 14 14:30:00 2026 +0530
 2 files changed, 28 insertions(+)
 # NOTICE: SHA changed from 4f8a192 to 7e9b2c3! It is a brand new commit!

# Inspecting Author vs Committer metadata:
$ git log -1 --format=fuller
commit 7e9b2c3a1b2c3d4e5f67890123456789abcdef01
Author:     Alice Engineer <alice@work.com>
AuthorDate: Sat Nov 14 14:30:00 2026 +0530
Commit:     Bob TechLead <bob@work.com>
CommitDate: Sat Nov 14 14:35:12 2026 +0530

    feat(api): add health check endpoint
```

---

## 7. Visual Explanation: The Conventional Commits Standard

Enterprise teams mein commits ka standard specification follow kiya jata hai:

```text
       ┌────────── Type: feat, fix, docs, chore, refactor, perf, test
       │      ┌─── Scope: Component / module name (optional)
       ▼      ▼
    feat(billing): implement stripe webhook retry handler
    │                                                    │
    └────────────────── Subject (≤ 50 chars) ────────────┘

    [Optional Body: Explain the WHY, not just the WHAT (≤ 72 chars/line)]
    Stripe webhooks occasionally fail during database lock contention.
    This patch introduces exponential backoff with a max retry count of 5.

    [Optional Footer: Issue tracker references / Breaking changes]
    Fixes: #PROJ-1429
    BREAKING CHANGE: Webhook callback signature now requires secret header.
```

---

## 8. Common Mistakes: Developers Kya Galti Karte Hain?

1. **Amending Pushed Commits:**
   - Problem: GitHub par push karne ke baad `git commit --amend` chalana aur fir agle push par panic hona jab Git push reject kar deta hai.
2. **The "Everything in One Commit" Antipattern:**
   - Committing 45 modified files with message `"updates"`. Ye code review karna impossible bana deta hai.
3. **Commit Message Subject Too Long:**
   - Writing 150-character paragraphs in the first line. Git tooling (GitHub UI, `git log --oneline`) 50-72 characters ke baad subject line ko truncate (`...`) kar deta hai.

---

## 9. Troubleshooting Guide: The 7-Step Diagnostic Framework

Agar galti se galat files `--amend` ho jayein:

```text
1. OBSERVE:    Developer kehta hai: "Maine galti se uncommitted sensitive config
               file ke sath pichle commit ko --amend kar diya!"

2. INSPECT:    Git Reflog inspect karein:
               `git reflog -n 5`

3. UNDERSTAND: Reflog saaf dikhayega:
               9999999 HEAD@{0}: commit (amend): feat: my feature
               1111111 HEAD@{1}: commit: feat: my feature (The pristine original!)

4. PLAN:       Branch pointer ko turant purane original commit (`1111111`) par
               wapas reset karein.

5. EXECUTE:    git reset --soft HEAD@{1}

6. VERIFY:     `git log -1` run karein; original commit restore ho chuka hoga,
               aur unwanted files staging area mein hongi jahan se unhe unstage
               kiya ja sakta hai.

7. PREVENT:    Amending se pehle hamesha `git status -s` aur `git diff --staged`
               se verify karein ki sirf wahi files staged hain jo intended hain.
```

---

## 10. Practice Lab: Atomic Commits & Amend Surgery 💻

Chaliye terminal par step-by-step execute karte hain:

```bash
# Step 1: Create sandbox
mkdir -p /tmp/git-commit-craft-lab && cd /tmp/git-commit-craft-lab
git init -b main

# Step 2: Create a feature file with a typo in commit message
echo "export const calculateTax = (amount) => amount * 0.18;" > tax.js
git add tax.js
git commit -m "feat(tax): calculate txa amount"
# Notice the typo: "txa" instead of "tax"!

# Note original commit hash:
ORIGINAL_SHA=$(git rev-parse HEAD)
echo "Original Commit SHA: $ORIGINAL_SHA"

# Step 3: Fix the message via amend
git commit --amend -m "feat(tax): calculate tax amount"
AMENDED_SHA=$(git rev-parse HEAD)
echo "Amended Commit SHA:  $AMENDED_SHA"

# Step 4: Verify that the SHA changed!
if [ "$ORIGINAL_SHA" != "$AMENDED_SHA" ]; then
  echo "CONFIRMED: Amend generated a completely new cryptographic commit object!"
fi

# Step 5: Prove the original commit is still alive in the object database!
git cat-file -t $ORIGINAL_SHA
# Output: commit
git cat-file -p $ORIGINAL_SHA
# It still exists with the old typo message! Git never deletes immediately!

# Step 6: Test Reflog Rescue
git reflog -n 3
# Notice: HEAD@{0} is amend, HEAD@{1} is original commit!

# Cleanup
cd ~ && rm -rf /tmp/git-commit-craft-lab
```

---

## 11. Challenge: Test Your Understanding 🧠 (Scenario-Based Problems)

### 📌 Scenario Challenge 1: The Accidental Amend of a Pushed Commit
> **Problem Statement:**
> Ek junior developer ne branch `feature/auth` par commit `C1` banaya aur GitHub par push kar diya.
> 10 minute baad usne ek choti typo fix ki aur chala diya:
> `git commit --amend --no-edit`
> 
> Ab jab usne `git push origin feature/auth` chalaya, toh Git ne push reject kar diya:
> `! [rejected] feature/auth -> feature/auth (non-fast-forward)`
> `error: failed to push some refs...`
> 
> Junior developer ne StackOverflow dekha aur `git push --force` chalane laga.
> 
> **Questions:**
> 1. Git ne push reject kyu kiya? (Remote commit DAG ke perspective se explain karein).
> 2. Agar team mein doosre developers bhi is branch par kaam kar rahe hon, toh `git push --force` unke local setup ko kaise destroy karega?
> 3. Safe alternative kya tha: Amend karne ke bajaye naya commit banana kyu zaroori tha?

<details>
<summary><b>💡 Click to Reveal Complete Solution & Walkthrough</b></summary>

#### Solution & Architectural Analysis:

**1. Why push was rejected:**
GitHub ke remote pointer par Commit `C1` tha.
Local machine par developer ne amend karke naya Commit `C2` bana diya (jiska parent `C1` nahi hai, balki `C1` ka parent hai).
Local Git ab remote Git se **diverge** ho chuka hai! Kyunki remote par `C1` hai jo local history mein exist nahi karta, Git fast-forward push reject kar deta hai taaki remote data accidentally overwrite na ho.

**2. The Danger of `--force` in Multi-Dev Teams:**
Agar developer `git push --force` chala deta hai:
- Remote server se Commit `C1` forcibly delete ho jayega aur `C2` replace ho jayega.
- Teammate jisne subah `git pull` kiya tha, uske local repo mein abhi bhi purana `C1` hoga.
- Jab teammate agla pull/push karega, uske terminal par duplicate commits aur ugly merge conflicts paida honge!

**3. The Golden Rule of Commits:**
**Once a commit is pushed to a shared remote, its history is public and frozen.**
Typo fix karne ke liye amend karne ke bajaye ek naya atomic commit banana chahiye tha:
```bash
git commit -m "fix(auth): fix typo in parameter name"
git push origin feature/auth
```
History completely clean, linear aur 100% conflict-free rehti!
</details>

---

### 📌 Scenario Challenge 2: Simulating Different Author and Committer
> **Problem Statement:**
> Aapko ek commit banana hai jisme:
> - Author: `Alice <alice@contributor.org>` (Code writer)
> - Committer: `Bob <bob@maintainer.org>` (You, the maintainer merging the code)
> 
> **Question:** Terminal CLI par bina interactive rebase ke single `git commit` command se Author metadata ko explicitly override kaise karenge?

<details>
<summary><b>💡 Click to Reveal Complete Solution & Walkthrough</b></summary>

#### Solution: `--author` Flag

```bash
# Committer identity current git config se auto-pick hogi (Bob):
git config user.name "Bob Maintainer"
git config user.email "bob@maintainer.org"

# Make commit with overridden author:
git commit --author="Alice Contributor <alice@contributor.org>" -m "feat: add feature by Alice"
```

#### Verification:
```bash
git log -1 --format=fuller
# Notice:
# Author: Alice Contributor <alice@contributor.org>
# Commit: Bob Maintainer <bob@maintainer.org>
```
</details>

---

### 📌 Scenario Challenge 3: Splitting a Giant Commit into Two Atomic Commits
> **Problem Statement:**
> Ek developer ne galti se ek single commit mein:
> 1. Ek critical security patch add kiya (`src/security.js`).
> 2. Aur sath mein pure project ka CSS style redesign commit kar diya (`src/styles.css`).
> 
> Lead ne bola: *"Security patch production mein turant deploy hona hai, par redesign abhi approve nahi hua. Is single commit ko tod kar 2 alag atomic commits banao!"*
> 
> **Question:** `git reset --soft HEAD~1` use karke is commit ko 2 alag commits mein surgically split kaise karenge?

<details>
<summary><b>💡 Click to Reveal Complete Solution & Walkthrough</b></summary>

#### Solution: Commit Splitting via Soft Reset

#### Step-by-Step Execution:
```bash
# Step 1: Undo the last commit but keep all modified files in Staging:
git reset --soft HEAD~1

# Step 2: Unstage everything back to working directory:
git reset HEAD

# Step 3: Stage and commit ONLY the security fix:
git add src/security.js
git commit -m "fix(security): patch authentication bypass vulnerability"

# Step 4: Stage and commit the styles redesign separately:
git add src/styles.css
git commit -m "style(ui): update landing page theme"
```

#### Verification:
```bash
git log -2 --oneline
```
Aapka giant unmaintainable commit do clean, atomic, production-ready commits mein split ho chuka hai!
</details>

---

## 12. Interview Questions (FAANG & Product Companies)

### Q1: What is the technical difference between an Author and a Committer in Git?
**Answer:**
- **Author:** The person who originally wrote the code changes. The `AuthorDate` reflects when the code was originally authored.
- **Committer:** The person who applied or committed the change to the repository. The `CommitDate` reflects when the commit was actually created or amended (e.g. during a rebase, cherry-pick, or maintainer patch application).

### Q2: Why does `git commit --amend` change the commit SHA-1 hash?
**Answer:** Git's commit SHA is a cryptographic hash calculated over the commit's root Tree hash, parent commit hash(es), author metadata, committer metadata, timestamp, and message. When you run `git commit --amend`, at least one of these components (message, files, committer timestamp) changes. Because of cryptographic hash properties, modifying even a single byte produces an entirely new SHA hash.

### Q3: What is the "Golden Rule of History Rewriting"?
**Answer:** **Never rewrite history (via amend, rebase, or reset) on commits that have already been pushed to a shared public or team branch.** Rewriting public history forces divergence across team members' local repositories, leading to merge conflicts, lost work, and broken CI pipelines. History rewriting should strictly be reserved for private local feature branches before sharing.

---

## 13. Team Lead Perspective: Real Teams Mein Iska Use

1. **Squash and Merge Policies on GitHub:**
   Junior developers feature branch par kaam karte waqt 20 messy commits bana dete hain ("fix typo", "tested again", "wip"). Tech Lead GitHub repository par **Squash and Merge** strategy enforce karta hai. Jab PR merge hoti hai, toh GitHub un 20 commits ko condense karke ek single clean, atomic Conventional Commit bana kar `main` branch mein inject karta hai.
2. **Mandatory GPG Commit Signing for SOC2 / ISO Compliance:**
   Kyunki koi bhi developer terminal par `git config user.email "ceo@company.com"` likh kar identity spoof kar sakta hai, Tech Lead enterprise security compliance ke liye **GPG Signed Commits** mandate karta hai. GitHub sirf un commits par green **Verified** badge dikhata hai jo developer ke cryptographic private key se signed hon.

---

### 🔗 Official Documentation References:
- [Git SCM - Changing the Last Commit](https://git-scm.com/book/en/v2/Git-Basics-Undoing-Things#_git_amend)
- [Conventional Commits 1.0.0](https://www.conventionalcommits.org/en/v1.0.0/)
- [GitHub Docs - Signing Commits with GPG](https://docs.github.com/en/authentication/managing-commit-signature-verification/signing-commits)

