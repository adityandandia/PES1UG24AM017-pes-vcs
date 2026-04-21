# PES-VCS Lab Report
## Building a Version Control System from Scratch

**Name:** Aditya Narayan Dandia 
**SRN:** PES1UG24AM017

---

## Phase 1: Object Storage Foundation

### Overview
Implemented content-addressable storage using SHA-256 hashing. Objects are stored with a type header (`blob <size>\0`) and sharded into subdirectories by the first 2 hex characters of their hash.

### Key Implementation Details
- **`object_write`**: Prepends type header, computes SHA-256, writes atomically using temp-file-then-rename pattern
- **`object_read`**: Reads object, parses header, verifies integrity by recomputing hash
- **Deduplication**: Identical content produces identical hashes, stored only once

### Screenshots

**Screenshot 1A: Object Storage Tests**
![Phase 1A - Test Output](Screenshots/1A.png)

**Screenshot 1B: Sharded Object Store Structure**
![Phase 1B - Object Directory](Screenshots/1B.png)

---

## Phase 2: Tree Objects

### Overview
Implemented tree serialization to represent directory structures. Trees contain entries with mode, hash, and filename, supporting nested paths.

### Key Implementation Details
- **`tree_from_index`**: Builds hierarchical tree structure from flat index
- Handles nested paths (e.g., `src/main.c` creates `src` subtree)
- Entries sorted deterministically for reproducible hashes

### Screenshots

**Screenshot 2A: Tree Tests Passing**
![Phase 2A - Test Output](Screenshots/2A.png)

**Screenshot 2B: Raw Tree Object Binary Format**
![Phase 2B - Binary Dump](Screenshots/2B.png)

---

## Phase 3: The Index (Staging Area)

### Overview
Implemented the staging area as a text-based file format. The index tracks which files are prepared for the next commit.

### Key Implementation Details
- **Index format**: `<mode> <hash> <mtime> <size> <path>` per line
- **`index_load`**: Parses text file into Index struct
- **`index_save`**: Atomic write with fsync() before rename
- **`index_add`**: Stages file by computing blob hash and updating index

### Screenshots

**Screenshot 3A: PES Status Output**
![Phase 3A - Status Output](Screenshots/3A.png)

**Screenshot 3B: Index File Contents**
![Phase 3B - Index Contents](Screenshots/3B.png)

---

## Phase 4: Commits and History

### Overview
Implemented commit creation and history traversal. Commits tie together trees with metadata (author, timestamp, message, parent).

### Key Implementation Details
- **`commit_create`**: Builds tree from index, reads HEAD for parent, writes commit object, updates branch ref
- **`commit_walk`**: Traverses parent chain to display history
- **Reference chain**: HEAD → refs/heads/main → commit hash

### Screenshots

**Screenshot 4A: Commit Log with Three Commits**
![Phase 4A - Log Output](Screenshots/4A.png)

**Screenshot 4B: Object Store Growth After Commits**
![Phase 4B - Object Files](Screenshots/4B.png)

**Screenshot 4C: Reference Chain (HEAD and main)**
![Phase 4C - Refs](Screenshots/4C.png)

**Final Screenshot: Test Integrations**  
![Final Screenshot](Screenshots/final.png)

---

## Phase 5: Branching and Checkout (Analysis)

### Q5.1: Implementing `pes checkout <branch>`

**Files that need to change in `.pes/`:**
1. **`.pes/HEAD`**: Update to point to the target branch reference file (e.g., change from `ref: refs/heads/main` to `ref: refs/heads/feature`)
2. **Working directory files**: Must be updated to match the target branch's tree snapshot

**What makes this operation complex:**

The complexity arises from several factors:

1. **Working directory synchronization**: After updating HEAD, every file in the working directory must be compared against the target tree. Files that differ need to be overwritten, new files need to be created, and files that exist in the current branch but not the target must be removed.

2. **Preserving uncommitted changes**: If the user has modified files that aren't committed, blindly overwriting them would lose work. The checkout must detect these changes and either preserve them or refuse the operation.

3. **Atomic operation**: The entire checkout should be atomic—either all files are updated successfully, or none are. Partial checkouts leave the repository in an inconsistent state.

4. **Path conflicts**: Files may have been renamed between branches, or a file in one branch might be a directory in another.

---

### Q5.2: Detecting "Dirty Working Directory" Conflicts

**Detection algorithm using only the index and object store:**

1. **Load the current index** (`.pes/index`) which contains the staged snapshot with mode, hash, mtime, size, and path for each file.

2. **Load the target branch's tree** by:
   - Reading the target branch ref file (e.g., `.pes/refs/heads/feature`)
   - Parsing the commit object to get its tree hash
   - Traversing the tree to get all file paths and their blob hashes

3. **For each file in the current index:**
   - Look up the same path in the target tree
   - If the path exists in the target tree with a **different hash**, compare the current working directory file's hash against the index hash
   - If the working directory hash differs from the index hash, the file has uncommitted changes
   - **Conflict detected**: The file has uncommitted changes AND differs between branches

4. **For files only in current branch** (not in target): These are safe—they'll be removed but represent no conflict.

5. **For files only in target branch**: These will be created fresh, no conflict possible.

**The key insight**: The index represents the "clean" state of the last commit/stage. By comparing working directory files against their index hashes, we detect modifications. If such a modified file also differs in the target branch, checkout must refuse.

---

### Q5.3: Detached HEAD State

**What happens when making commits in detached HEAD:**

When HEAD contains a commit hash directly instead of `ref: refs/heads/<branch>`, you're in a "detached HEAD" state. Commits made in this state:

1. **Are created normally**: The commit object is written to `.pes/objects/`, with its parent pointing to the detached commit
2. **Have no branch reference**: No file in `.pes/refs/heads/` points to these commits
3. **Are only reachable from the detached HEAD position**: Once you checkout a different branch or commit, these commits become unreachable

**Recovery:**

The commits aren't immediately lost because:
- The reflog (if implemented) records all previous HEAD positions
- The object still exists in `.pes/objects/` until garbage collection

To recover:
1. Find the commit hash (from reflog: `git reflog`, or from object store inspection)
2. Create a new branch pointing to it: `git branch recovered <hash>`
3. Or checkout directly: `git checkout <hash>` (returns to detached state)

**Without a reflog**, recovery requires manually inspecting object timestamps or using `git fsck --lost-found` to find dangling commits.

---

## Phase 6: Garbage Collection and Space Reclamation (Analysis)

### Q6.1: Algorithm to Find and Delete Unreachable Objects

**Algorithm:**

1. **Start from all roots** (reachable references):
   - All branch refs in `.pes/refs/heads/`
   - All tags in `.pes/refs/tags/`
   - The current HEAD (if detached)
   - The reflog entries (if preserving recent unreachable commits)

2. **Breadth-first traversal** using a queue:
   ```
   reachable = empty set
   queue = all root commits from refs

   while queue not empty:
       commit = queue.dequeue()
       if commit in reachable: continue
       reachable.add(commit)

       parse commit to get:
           - tree hash → queue.enqueue(tree)
           - parent hash → queue.enqueue(parent)

   process_tree(tree_hash):
       parse tree to get all entry hashes
       for each entry:
           if entry is blob: reachable.add(blob)
           if entry is tree: queue.enqueue(tree)
   ```

3. **Scan all objects** in `.pes/objects/`:
   - For each object hash not in `reachable` set, mark for deletion

4. **Delete unreachable objects**

**Data structure for efficiency:**

A **hash set** (O(1) lookup) is ideal for tracking reachable hashes. For 100,000 commits with 50 branches:

- **Objects to visit**: 
  - 100,000 commits (if all reachable from branches)
  - ~100,000 trees (one per commit, roughly)
  - Variable blobs (depends on file changes per commit)
  
- **Estimated total**: 300,000 - 500,000 objects for a moderately active repository

The hash set provides O(1) membership testing, making the final scan efficient even with hundreds of thousands of objects.

---

### Q6.2: Race Condition with Concurrent GC and Commit

**The race condition:**

```
Time    Commit Operation              GC Operation
----    --------------              ------------
T1      Read HEAD to get parent
T2                                  Start reachability analysis
T3      Build tree from index
T4                                  Complete analysis, identify
        (tree hash: abc123)          unreachable objects
T5      Write commit object           Delete "unreachable" abc123
        (points to tree abc123)      (not yet referenced by any commit)
T6      Update branch ref
```

**The problem**: GC at T4 sees tree `abc123` as unreachable because no commit references it yet (commit written at T5). GC deletes it. Commit at T6 creates a commit pointing to a now-missing tree, corrupting the repository.

**How Git's real GC avoids this:**

1. **Two-phase GC**: Git uses `git gc --auto` which runs `git repack` and `git prune` separately. Objects are never deleted immediately.

2. **Grace period**: Git's `git prune` only deletes objects older than `gc.pruneExpire` (default: 2 weeks). This ensures any in-progress operation has time to complete.

3. **Lock files**: Git uses `.git/index.lock` and similar mechanisms to prevent concurrent modifications to critical files.

4. **Pack files**: `git repack` consolidates objects into pack files atomically. The old pack is only deleted after the new one is fully written and verified.

5. **Reference transactions**: Modern Git uses reference transactions to ensure ref updates are atomic and consistent.

**The key insight**: GC should only delete objects that have been unreachable for a sufficiently long time, not objects that became unreachable in the current instant.

---
