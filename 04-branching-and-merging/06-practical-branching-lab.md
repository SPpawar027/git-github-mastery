# Chapter 06: Capstone Practical Lab (Module 04 Branching & Merging Mastery)

---

## 1. Concept: Ye Kya Hai?

Ye Module 04 ka **Capstone Hands-on Lab** hai.
Pichle 5 chapters mein aapne Branching aur Merging ke saare foundational aur advanced concepts padhe hain:
- Branches as 41-byte pointers
- Enterprise Branching Strategies (Git Flow vs Trunk-Based)
- 3-Way Merge vs Rebase
- Fast-Forward vs `--no-ff`
- Surgical Cherry-Picking with `-x` audit trace

Is lab mein hum ek realistic **Sprint Simulation** execute karenge jahan aap ek sath ek naya feature develop karenge, ek emergency production hotfix handle karenge, aur complex rebase/cherry-pick operations ko bina kisi confusion ke terminal par live perform karenge.

---

## 2. Why: Iski Zarurat Kyu Padi?

Theory jaan lena kafi nahi hota. Real development team mein pressure tab aata hai jab:
- Feature branch par kaam chal raha ho aur achanak hotfix aa jaye.
- Merge conflict aane par terminal freeze ho jaye.
- Team Lead bole ki merge commit mat banao, rebase karke clean history do.

Jab tak aap in saare scenarios ko ek single cohesive terminal workflow mein apne haath se execute nahi karenge, aap production incidents ke waqt confident nahi ho sakte.

---

## 3. Real-World Scenario: The FinTech Sprint Drill

Aap ek FinTech payments API repository maintain kar rahe hain:
1. `main` branch production release `v1.0.0` par khadi hai.
2. Team ek naya feature bana rahi hai: `feature/recurring-payments`.
3. Development ke dauran production par ek critical security bug report hota hai: `hotfix/cve-token-leak`.
4. Aapko hotfix ko `main` par merge karna hai, release `v1.0.1` tag karna hai, use `feature/recurring-payments` branch par cherry-pick/rebase karna hai, aur feature branch ko rebase karke clean state mein merge karna hai.

---

## 4. Internal Working: Lab State Map

```text
Phase 1: Bootstrap Repo & v1.0.0 Release Tag
Phase 2: Create feature/recurring-payments & make WIP commits
Phase 3: Emergency Interruption -> Create hotfix/cve-token-leak from main
Phase 4: Merge hotfix into main with --no-ff and tag v1.0.1
Phase 5: Rebase feature branch on top of updated main
Phase 6: Surgical Cherry-Pick verification & final cleanup
```

---

## 5. Commands Reference for this Lab

| Command | Safety | Purpose in Lab |
|---|:---:|---|
| `git init -b main` | `🟡 [CAUTION]` | Repo bootstrap with standard main |
| `git switch -c <name>` | `🟡 [CAUTION]` | Atomic branch create & switch |
| `git commit -am "..."` | `🟡 [CAUTION]` | Fast atomic tracked commits |
| `git merge --no-ff <name>` | `🟡 [CAUTION]` | Enforce explicit merge commit |
| `git rebase main` | `🔴 [DANGEROUS]` | Linear history replay |
| `git cherry-pick -x <sha>` | `🟡 [CAUTION]` | Surgical backporting |
| `git log --graph --oneline --all` | `🟢 [SAFE]` | Full DAG visual inspection |

---

## 6. Step-by-Step Practical Lab Execution 💻

Har step ko apne Linux ya macOS terminal par sequence mein run karein:

### Phase 1: Bootstrap & Production v1.0.0 Release
```bash
# Step 1.1: Create isolated lab directory
mkdir -p ~/git-labs/branching-mastery && cd ~/git-labs/branching-mastery
git init -b main

# Step 1.2: Create initial production codebase
cat << 'EOF' > server.js
// FinTech Core Payment Gateway v1.0.0
function processTransaction(account, amount) {
    console.log("Processing standard transaction:", amount);
    return true;
}
EOF
git add server.js
git commit -m "feat(core): initial production release v1.0.0"

# Step 1.3: Tag official release v1.0.0
git tag -a v1.0.0 -m "Production Release v1.0.0"
```

---

### Phase 2: Feature Branch Development
```bash
# Step 2.1: Branch out for the recurring payments feature
git switch -c feature/recurring-payments

# Step 2.2: Implement recurring logic
cat << 'EOF' >> server.js

function processRecurring(account, amount, interval) {
    console.log("Setting up subscription interval:", interval);
    return processTransaction(account, amount);
}
EOF
git commit -am "feat(subscription): add recurring billing handler"

# Step 2.3: Add documentation commit
echo "# Subscription API Documentation" > docs.md
git add docs.md
git commit -m "docs(subscription): add initial api specs"

# Verify feature branch has 2 commits ahead of main:
git log --oneline -3
```

---

### Phase 3: Emergency Hotfix Interruption
Achanak production security alert aaya! Developer ko feature branch chhod kar turant `main` par switch karna hai:

```bash
# Step 3.1: Switch to main (Working tree is clean, so switch is instant!)
git switch main

# Step 3.2: Cut emergency hotfix branch directly from main
git switch -c hotfix/cve-token-leak

# Step 3.3: Patch the security vulnerability in server.js
cat << 'EOF' > server.js
// FinTech Core Payment Gateway v1.0.1 (SECURITY PATCHED)
function sanitizeToken(token) {
    if (!token) throw new Error("Invalid Auth Token");
    return token.trim();
}

function processTransaction(account, amount) {
    console.log("Processing standard transaction:", amount);
    return true;
}
EOF
git commit -am "fix(security): patch CVE auth token validation vulnerability"
HOTFIX_SHA=$(git rev-parse HEAD)
echo "Hotfix Commit SHA: $HOTFIX_SHA"
```

---

### Phase 4: Production Hotfix Release & Tagging
```bash
# Step 4.1: Switch back to main
git switch main

# Step 4.2: Merge hotfix with explicit --no-ff merge commit
git merge --no-ff hotfix/cve-token-leak -m "Merge hotfix/cve-token-leak into main (v1.0.1)"

# Step 4.3: Tag production v1.0.1
git tag -a v1.0.1 -m "Production Hotfix Release v1.0.1"

# Step 4.4: Safely delete the hotfix branch
git branch -d hotfix/cve-token-leak
```

---

### Phase 5: Rebase Feature Branch on Updated Main
Ab feature branch ko `main` ke naye commits (v1.0.1 security fix) ke sath synchronize karna hai bina messy merge commits ke:

```bash
# Step 5.1: Switch to feature branch
git switch feature/recurring-payments

# Step 5.2: Rebase on top of main!
git rebase main

# If Git pauses for conflict in server.js:
# We resolve by keeping BOTH the security patch AND the recurring billing function!
cat << 'EOF' > server.js
// FinTech Core Payment Gateway v1.0.1 (SECURITY PATCHED)
function sanitizeToken(token) {
    if (!token) throw new Error("Invalid Auth Token");
    return token.trim();
}

function processTransaction(account, amount) {
    console.log("Processing standard transaction:", amount);
    return true;
}

function processRecurring(account, amount, interval) {
    console.log("Setting up subscription interval:", interval);
    return processTransaction(account, amount);
}
EOF

# Step 5.3: Stage the resolution and continue rebase
git add server.js
git rebase --continue

# Step 5.4: Verify rebase succeeded
git status
# Notice: Working tree clean, successfully rebased!
```

---

### Phase 6: Final Integration & Full DAG Inspection
```bash
# Step 6.1: Switch to main
git switch main

# Step 6.2: Merge the rebased feature branch (Clean fast-forward or clean merge!)
git merge --no-ff feature/recurring-payments -m "feat(billing): merge recurring payments feature"

# Step 6.3: Delete feature branch
git branch -d feature/recurring-payments

# Step 6.4: THE MOMENT OF TRUTH: Inspect the beautiful DAG!
git log --graph --oneline --all --decorate
```

Expected Terminal Output:
```text
*   7a2b3c1 (HEAD -> main) feat(billing): merge recurring payments feature
|\  
| * e4f5a6b docs(subscription): add initial api specs
| * c3d4e5f feat(subscription): add recurring billing handler
|/  
*   9f8e7d6 (tag: v1.0.1) Merge hotfix/cve-token-leak into main (v1.0.1)
|\  
| * 1a2b3c4 fix(security): patch CVE auth token validation vulnerability
|/  
* 8e20986 (tag: v1.0.0) feat(core): initial production release v1.0.0
```

---

## 7. Visual Summary of the Final DAG Architecture

```text
v1.0.0 ───► [Hotfix CVE] ────────► [v1.0.1 Merge Commit]
                  \                         ▲
                   \─── (Rebased on top) ───┤
                                            ├──► [Recurring Logic] ──► [Docs] ──► [Feature Merge] ◄── HEAD (main)
```

---

## 8. Common Mistakes in Lab

1. **Rebasing while in Detached HEAD:**
   Hamesha verify karein ki aap `feature/recurring-payments` branch par hain rebase chalane se pehle.
2. **Running `git commit` instead of `git rebase --continue`:**
   Conflict resolve karne ke baad `git add` karein, par `git commit` mat chalayein! Hamesha `git rebase --continue` use karein.

---

## 9. Troubleshooting Scenario: Rebase Abort Drill

Agar rebase ke dauran conflict itna ulajh jaye ki aap scratch se shuru karna chahein:
```bash
# Safe emergency exit:
git rebase --abort
```
Repo 100% pre-rebase state par wapas chali jayegi bina kisi data loss ke!

---

## 10. Lab Completion Challenge 🧠 (Scenario-Based Problems)

### 📌 Scenario Challenge 1: The Maintenance Release Backport Simulation
> **Mastery Task:**
> 1. Ek nayi branch kaatein `release/v1.0.x` tag `v1.0.0` se (simulate karna hai purana enterprise client jo v1.0.1 par upgrade nahi kar sakta).
> 2. `git cherry-pick -x $HOTFIX_SHA` use karke sirf CVE hotfix ko `release/v1.0.x` branch par backport karein.
> 3. Verify karein ki commit message ke footer mein `(cherry picked from commit ...)` automatic audit trace generate hua ya nahi!

<details>
<summary><b>💡 Click to Reveal Complete Solution & Walkthrough</b></summary>

#### Step-by-Step Solution:

```bash
# Step 1: Create maintenance branch from v1.0.0 tag:
git switch -c release/v1.0.x v1.0.0

# Step 2: Cherry-pick the hotfix with -x:
git cherry-pick -x $HOTFIX_SHA

# Step 3: Inspect the log:
git log -1
# Output will display the exact audit trail footer!
# Tag the maintenance release:
git tag -a v1.0.0-patch1 -m "Backported CVE patch to v1.0.x"
```
Aapne real-world enterprise multi-version maintenance drill master kar liya!
</details>

---

## 11. Module 04 Self-Assessment Checklist

Bina solution dekhe in operations ko terminal par demonstrate karein:
- [ ] Kya aap Fast-Forward aur `--no-ff` merge ke visual difference ko explain kar sakte hain?
- [ ] Kya aap rebase conflict ko resolve karke `--continue` se aage badh sakte hain?
- [ ] Kya aap `git cherry-pick -x` se single commit ko audit trace ke sath transplant kar sakte hain?
- [ ] Kya aap `git rebase --abort` se kisi bhi broken rebase se bina data loss ke exit le sakte hain?

Agar haan, toh **Congratulations!** Aapne **Module 04: Branching and Merging Mechanics** ko fully master kar liya hai. 
Ab aap [PROGRESS-TRACKER.md](../PROGRESS-TRACKER.md) mein Module 04 ke saare chapters ko `[*]` (Mastered) mark kar sakte hain aur **Module 05: GitHub and Authentication Architecture** par aage badh sakte hain!

