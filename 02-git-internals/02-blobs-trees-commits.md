# Chapter 02: Blobs, Trees & Commits (The Holy Trinity of Git Internals)

---

## 1. Concept: Ye Kya Hai?

Git ke andar aapka project koi mysterious black box nahi hai. Git aapke pure operating system folder structure ko sirf 3 objects ke combination se represent karta hai:

1. **Blob:** File ka raw data (No metadata, no name, no permissions).
2. **Tree:** Ek directory listing (maps filenames aur permissions ko Blobs ya doosre Subtrees ke sath).
3. **Commit:** Root tree ka snapshot + parent pointer + author metadata.

In teeno ke relation ko **The Holy Trinity of Git Internals** kaha jaata hai. Jab aap in teeno ka relation samajh lete hain, toh aap command-line porcelain (`git add`, `git commit`) ke mohtaaj nahi rehte—aap Git ko low-level engine ki tarah operate kar sakte hain.

---

## 2. Why: Iski Zarurat Kyu Padi? (Why Can't Git Track Empty Folders?)

Aksar developers poochte hain:
*"Maine project mein ek empty folder banaya `assets/temp/`, lekin `git status` mein wo dikh hi nahi raha hai! Git empty directory track kyu nahi karta?"*

### The Architectural Reason:
Git ke data model ko dekhein:
- Git sirf **Blobs** ko track karta hai (jo files ke content se bante hain).
- **Trees** tab bante hain jab unke andar kam se kam ek Blob ya Subtree ho.
- Agar ek folder bilkul empty hai, toh uske andar koi file nahi hai -> Koi Blob nahi bana -> Koi Tree entry generate nahi ho sakti!
- Isiliye Git empty folders ko track nahi kar sakta.
- **Industry Solution:** Developers empty folder ke andar `.gitkeep` (ya koi dummy file) bana dete hain taaki Git ko kam se kam ek 0-byte blob mil sake tree entry banane ke liye.

---

## 3. Real-World Example: Multi-Tier Nested Directory Structure

Maan lijiye aapka enterprise backend repo is tarah ka hai:

```text
backend-service/
├── package.json
└── src/
    ├── controllers/
    │   └── auth.js
    └── server.js
```

Git ise internally kaise map karta hai?

```text
                    [Commit Object]
                           │
                           ▼
                  [Root Tree Object]
                  ├── 100644 blob (package.json)
                  └── 040000 tree (src/)
                               │
                               ▼
                        [src/ Tree Object]
                        ├── 100644 blob (server.js)
                        └── 040000 tree (controllers/)
                                     │
                                     ▼
                            [controllers/ Tree Object]
                            └── 100644 blob (auth.js)
```

Notice karein: Har folder ek **Tree Object** hai! Agar sirf `auth.js` change hota hai, toh sirf `controllers/` tree, `src/` tree aur Root tree ke hash change honge; `package.json` aur `server.js` ke Blobs 100% reuse honge!

---

## 4. Internal Working: Handcrafting a Commit (Plumbing vs Porcelain)

High-level command `git commit` asal mein 3 low-level commands ka wrapper hota hai:

1. **`git update-index`:** Working directory ki files ko hash karke Blobs banata hai aur unhe Staging Area (`.git/index`) mein load karta hai. (Ye kaam `git add` karta hai).
2. **`git write-tree`:** Current Staging Area se ek **Root Tree object** create karta hai aur uska SHA-1 hash return karta hai.
3. **`git commit-tree <tree-sha> -m "message"`:** Us Tree hash ko le kar ek **Commit object** banata hai jo parent commit ko point karta hai.
4. **`git update-ref refs/heads/main <commit-sha>`:** Active branch ke pointer ko is naye commit SHA par move kar deta hai.

---

## 5. Commands: Inspecting Trees and Low-Level Commit Creation

| Command | Safety | Description |
|---|:---:|---|
| `git ls-tree <tree-ish>` | `🟢 [SAFE]` | Tree object ke direct contents (files aur subdirectories) list karta hai. |
| `git ls-tree -r <tree-ish>` | `🟢 [SAFE]` | Recursive list: pure repository ke saare files aur unke blob hashes dikhata hai. |
| `git ls-tree -r -t <tree-ish>` | `🟢 [SAFE]` | Saare Blobs ke sath beech ke intermediate Tree objects ko bhi print karta hai. |
| `git write-tree` | `🟡 [CAUTION]` | Index (Staging area) ka ek immutable Tree object create karta hai. |
| `git commit-tree <tree-sha> -m "..."` | `🟡 [CAUTION]` | Tree hash se manually ek Commit object create karta hai. |
| `git update-ref <ref> <commit-sha>` | `🟡 [CAUTION]` | Kisi branch reference ko naye commit par manually update karta hai. |

---

## 6. Expected Output: Terminal Par Tree Inspection

```bash
# Listing root tree of the latest commit:
$ git ls-tree HEAD
100644 blob 3b18e512dba79e4c8300dd08aeb37f8e728b8dad	.gitignore
100644 blob f9b4c09d57a22ae7cb289b4f2c8d234a41300a7b	README.md
040000 tree 8e20986756f34e2ab69e46a9a089025e16544c00	src

# Recursive inspection showing full paths:
$ git ls-tree -r HEAD
100644 blob 3b18e512dba79e4c8300dd08aeb37f8e728b8dad	.gitignore
100644 blob f9b4c09d57a22ae7cb289b4f2c8d234a41300a7b	README.md
100644 blob 5e81d77a28e32c4e43029f6356e72b43b6702e77	src/index.js
100644 blob 912384a8b72d2f3c091928374a56b82910293847	src/server.js
```

---

## 7. Visual Explanation: The Tree-Blob Resolution Mechanism

```text
Octal Mode   Object Type    SHA-1 Object Hash                           Path
──────────   ───────────    ────────────────────────────────────────    ────────────
100644       blob           f9b4c09d57a22ae7cb289b4f2c8d234a41300a7b    README.md
040000       tree           8e20986756f34e2ab69e46a9a089025e16544c00    src/
                               │
                               ▼
              100644 blob 5e81d77a28e3...  src/index.js
              100644 blob 912384a8b72d...  src/server.js

* 100644 = Regular non-executable file
* 100755 = Executable file (script/binary)
* 120000 = Symbolic link
* 040000 = Directory (Tree)
```

---

## 8. Common Mistakes: Developers Kya Galti Karte Hain?

1. **Expecting Empty Folders to be Committed:**
   - Problem: Empty directory banakar GitHub par push karna aur complain karna ki folder push nahi hua.
   - Fix: Always put a `.gitkeep` file inside the folder.
2. **Confusing Blobs with Commits:**
   - Problem: Sochna ki `git checkout` kisi blob par switch karta hai.
   - Reality: Git checkout ya switch hamesha **Commits** ya **Branches** par switch karta hai. Blobs sirf file fragments hote hain.

---

## 9. Troubleshooting Guide: The 7-Step Diagnostic Framework

Agar kisi directory mein permission changes track na ho rahe hon:

```text
1. OBSERVE:    Aapne script executable banayi: `chmod +x deploy.sh`
               Lekin `git status` koi change detect nahi kar raha.

2. INSPECT:    Git configuration mein filemode setting check karein:
               `git config core.filemode`

3. UNDERSTAND: Windows systems par ya kuch specific Linux mounts par
               `core.filemode` false set ho jata hai, jisse Git file permissions
               (100644 vs 100755) track karna ignore kar deta hai.

4. PLAN:       `core.filemode` ko true enable karein ya manual update-index karein.

5. EXECUTE:    git config core.filemode true
               git update-index --chmod=+x deploy.sh

6. VERIFY:     `git diff --staged` run karein; output dikhayega:
               "old mode 100644, new mode 100755"

7. PREVENT:    Ensure karein ki deployment scripts create karte hi `chmod +x`
               karke commit kiye jayein aur CI build environment match kare.
```

---

## 10. Practice Lab: Handcrafting a Commit Using Only Plumbing Commands 💻

Is mind-blowing lab mein hum **bina `git add` ya `git commit` chalaye** Git ke low-level plumbing commands se ek official, valid commit banayenge:

```bash
# Step 1: Create fresh sandbox
mkdir -p /tmp/git-handcrafted-commit && cd /tmp/git-handcrafted-commit
git init -b main

# Step 2: Create a physical file
echo "Enterprise Plumbing Architecture" > architecture.txt

# Step 3: Hash the file and create a blob in .git/objects/
BLOB_SHA=$(git hash-object -w architecture.txt)
echo "Generated Blob SHA: $BLOB_SHA"

# Step 4: Add the blob directly into Git's Staging Index without git add!
git update-index --add --cacheinfo 100644 $BLOB_SHA architecture.txt
git status -s
# Output shows: A  architecture.txt (It is STAGED!)

# Step 5: Write the Staging Index into a Tree Object
TREE_SHA=$(git write-tree)
echo "Generated Tree SHA: $TREE_SHA"

# Verify Tree contents:
git cat-file -p $TREE_SHA
# Output shows: 100644 blob <BLOB_SHA> architecture.txt

# Step 6: Create a Commit Object pointing to this Tree without git commit!
COMMIT_SHA=$(echo "feat: manually handcrafted plumbing commit" | git commit-tree $TREE_SHA)
echo "Generated Commit SHA: $COMMIT_SHA"

# Verify Commit Object:
git cat-file -p $COMMIT_SHA
# Notice: tree <TREE_SHA>, author, committer, and commit message!

# Step 7: Update our 'main' branch reference to point to this new commit!
git update-ref refs/heads/main $COMMIT_SHA

# Step 8: THE MOMENT OF TRUTH: Verify via high-level porcelain!
git log --oneline
git status
# Boom! Working tree is clean, and your commit is permanently in the history!

# Cleanup
cd ~ && rm -rf /tmp/git-handcrafted-commit
```

---

## 11. Challenge: Test Your Understanding 🧠 (Scenario-Based Problems)

### 📌 Scenario Challenge 1: Constructing a Parent-Child DAG Chain via Plumbing
> **Problem Statement:**
> Pichle lab mein humne pehla root commit manually plumbing se banaya tha.
> Ab aapko ek doosra commit manually banana hai low-level commands se:
> 1. Ek nayi file add karein `notes.txt`.
> 2. `git write-tree` se naya tree banayein.
> 3. `git commit-tree` run karte waqt `-p <parent-commit-sha>` flag pass karein taaki naya commit purane commit ko apna **Parent** declare kare!
> 4. `git update-ref` chala kar `main` branch ko naye commit par point karein.
> 5. `git log --graph` chala kar dekhein ki kya Git bina kisi issue ke 2 commits ki linear chain dikhata hai.

<details>
<summary><b>💡 Click to Reveal Complete Solution & Walkthrough</b></summary>

#### Step-by-Step Plumbing Execution:

```bash
# Assuming you are in the repo with previous commit:
PARENT_COMMIT=$(git rev-parse HEAD)
echo "Parent Commit SHA: $PARENT_COMMIT"

# Step 1: Create new file and hash it
echo "Plumbing Part 2: Parent Pointers" > notes.txt
NEW_BLOB=$(git hash-object -w notes.txt)

# Step 2: Add to staging index
git update-index --add --cacheinfo 100644 $NEW_BLOB notes.txt

# Step 3: Write new tree
NEW_TREE=$(git write-tree)
echo "New Tree SHA: $NEW_TREE"

# Step 4: Create commit with explicit parent flag (-p)
CHILD_COMMIT=$(echo "feat: second commit linked via parent pointer" | git commit-tree $NEW_TREE -p $PARENT_COMMIT)
echo "Child Commit SHA: $CHILD_COMMIT"

# Step 5: Update the branch reference
git update-ref refs/heads/main $CHILD_COMMIT

# Step 6: Verify the DAG graph
git log --oneline --graph
```

#### Expected Output:
```text
* a1b2c3d (HEAD -> main) feat: second commit linked via parent pointer
* 5e81d77 feat: manually handcrafted plumbing commit
```
Aapne bina `git commit` use kiye Git ka authentic Parent-Child Directed Acyclic Graph (DAG) physically form kar diya!
</details>

---

### 📌 Scenario Challenge 2: The Mysterious "Empty Directory" PR Rejection
> **Problem Statement:**
> Ek frontend developer ne React project mein images ke liye folder banaya:
> `mkdir -p src/assets/images`
> 
> Usne `git add .` chalaya, `git commit` kiya, aur GitHub par PR open kar di.
> PR review mein Lead ne dekha ki `src/assets/images` folder PR diff mein exist hi nahi karta!
> Junior ne bola: *"Maine toh terminal mein folder banaya tha aur commit bhi kiya tha!"*
> 
> **Questions:**
> 1. Git ne folder ko commit kyu nahi kiya? (Tree aur Blob data structures ke perspective se samjhayein).
> 2. Industry standard trick kya hai is folder ko repo mein persist karne ke liye?

<details>
<summary><b>💡 Click to Reveal Complete Solution & Walkthrough</b></summary>

#### Solution & Explanation:

1. **Why Git Ignored the Folder:**
   Git ka internal object model sirf **Blobs** (files) track karta hai.
   Tree object tab banta hai jab directory ke andar koi file ho. 
   Agar folder empty hai, toh koi Blob exist nahi karta -> Index mein koi file hash nahi aata -> `git write-tree` us folder ke liye koi subtree entry create nahi kar sakta!

2. **The Solution (`.gitkeep`):**
   Industry standard solution hai ki empty directory ke andar ek zero-byte file create ki jaye:
   ```bash
   touch src/assets/images/.gitkeep
   git add src/assets/images/.gitkeep
   git commit -m "chore: keep images directory structure"
   ```
   Ab Git ke paas ek Blob (even if 0 bytes: `e69de2...`) ban jata hai, aur `images/` folder ka Tree object generate ho jata hai!
</details>

---

## 12. Interview Questions (FAANG & Product Companies)

### Q1: Why does `git write-tree` fail if there are unmerged conflicts in the Index?
**Answer:** `git write-tree` Staging Area (`.git/index`) se direct Tree object generate karta hai. Agar merge conflict ke dauran Index mein higher-order stage entries (stages 1, 2, 3) exist karti hain, toh Index ek ambiguous state mein hota hai. Git tab tak valid Tree hash produce nahi kar sakta jab tak conflict resolve karke Index stage 0 par normalize na ho jaye.

### Q2: What are the file permission modes supported by Git Tree objects?
**Answer:** Git traditional POSIX full permissions (like 644, 777, 755) store nahi karta. Git sirf 4 specific modes support karta hai:
1. `100644`: Normal non-executable file.
2. `100755`: Executable file.
3. `120000`: Symbolic link.
4. `160000`: Git Submodule (gitlink).
5. `040000`: Directory (Tree).

### Q3: How does Git detect file renames if Tree objects store filenames?
**Answer:** Git file renames ko explicit metadata ke roop mein store **nahi** karta. Jab commit A aur commit B ko compare kiya jata hai, agar commit A mein se file `old.js` gayab hai aur commit B mein `new.js` aa gayi hai, toh Git unke Blob hashes compare karta hai. Agar Blob hash exact match karta hai (ya content similarity score 50%+ hota hai), toh Git dynamically report karta hai: `renamed: old.js -> new.js`.

---

## 13. Team Lead Perspective: Real Teams Mein Iska Use

1. **Executable Permissions in CI/CD:**
   Aksar developers Windows machine par bash script (`deploy.sh`) likhte hain aur push kar dete hain. Windows par execute bit nahi hota, toh Git Tree mein mode `100644` chala jata hai. Jab Linux CI/CD runner script execute karta hai, toh fail ho jata hai: `permission denied: ./deploy.sh`. Tech Lead onboarding rules mein mandate karta hai: `git update-index --chmod=+x deploy.sh` taaki repo level par mode `100755` commit ho.
2. **Submodules vs Monorepo (Mode 160000):**
   Tree object mein mode `160000` ek Submodule ko represent karta hai. Tech Lead ko decision lena hota hai ki multi-repo architecture mein submodules use kiye jayein ya modern tooling ke sath single Monorepo approach apnayi jaye, kyunki submodules ke detached pointers team friction create karte hain.

---

### 🔗 Official Documentation References:
- [Git SCM - Git Internals: Tree Objects](https://git-scm.com/book/en/v2/Git-Internals-Git-Objects#_tree_objects)
- [Git SCM - git-write-tree Documentation](https://git-scm.com/docs/git-write-tree)
- [Git SCM - git-commit-tree Documentation](https://git-scm.com/docs/git-commit-tree)

