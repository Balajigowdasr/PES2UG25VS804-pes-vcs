PES-VCS: Version Control System Report
Name: Balaji Gowda S R

SRN: PES2UG25CS804

Section: A

Course: Operating Systems (Unit 4 – Orange Program Assignment)

🚀 Project Overview
PES-VCS is a specialized version control system built from the ground up in C. It implements the core logic of Git, including content-addressable storage, recursive tree structures, a staging area (index), and a commit history linked via parent pointers. The project explores fundamental Operating System concepts such as file descriptors, atomic writes (temp-file-then-rename), and filesystem sharding.

🏗️ Phase 1: Object Storage Foundation
In this phase, we implemented a content-addressable object store. Every piece of data (blob, tree, or commit) is stored as an object named by its SHA-256 hash.

Key Implementation Details:

object_write: Prepends a type header, hashes the content, and writes the file using a temporary file followed by a rename() call to ensure atomicity.
Directory Sharding: Objects are stored in subdirectories named after the first 2 hex characters of their hash (e.g., .pes/objects/a1/...) to prevent directory performance degradation.
object_read: Reads objects and performs an integrity check by re-hashing the data and comparing it to the identifier.
Phase 1 Verification
📸 Screenshot 1A:<img width="912" height="180" alt="1A" src="https://github.com/user-attachments/assets/012f9795-9ab5-4628-8c71-db0e0026e90b" />
 Output of ./test_objects
Phase 1 Tests

📸 Screenshot 1B: <img width="869" height="88" alt="1B" src="https://github.com/user-attachments/assets/d540f0ad-7a19-43d5-9fc7-8a23e0cb19d5" />
find .pes/objects -type f showing sharded structure
Object Sharding

🌲 Phase 2: Tree Objects
Phase 2 focused on directory representation. Unlike blobs (which store file contents), trees store directory listings—effectively mapping filenames to hashes and modes.

Key Implementation Details:

Recursive Tree Construction: Implemented tree_from_index to transform a flat list of staged files into a hierarchical tree of objects.
Binary Format: Created a custom binary serialization format for trees that allows for fast parsing and deterministic hashing.
Phase 2 Verification
📸 Screenshot 2A: <img width="1614" height="975" alt="2a" src="https://github.com/user-attachments/assets/8fb44783-e439-4ca3-91de-3320e83b21bf" />
Output of ./test_tree
Phase 2 Tests

📸 Screenshot 2B:<img width="827" height="111" alt="2b" src="https://github.com/user-attachments/assets/7d9e0125-687f-45c4-ba7c-ae616e99e28e" />
 xxd view of a raw tree object
Raw Tree Format

📑 Phase 3: Staging Area (The Index)
The index acts as the "preparation area" for commits. We implemented a human-readable, text-based index to track staged files and their metadata.

Key Implementation Details:

Text-Based Storage: The .pes/index file is stored as plain text with the format: <mode> <hash-hex> <timestamp> <size> <path>.
index_add: Computes the blob hash of a file, writes it to the store, and updates the index entry with the latest filesystem stat metadata.
Phase 3 Verification
📸 Screenshot 3A: <img width="1668" height="943" alt="3a" src="https://github.com/user-attachments/assets/9d68a21f-0a4f-459f-9f25-83494a99068a" />
pes init → pes add → pes status sequence
Index Status

📸 Screenshot 3B: <img width="1774" height="887" alt="3b" src="https://github.com/user-attachments/assets/d356ae84-3c07-4a29-a6a0-048caf1d975f" />
cat .pes/index showing human-readable content
Index Content

💾 Phase 4: Commits and History
This final implementation phase ties everything together into a commit object, creating a snapshot of the project at a specific point in time.

Key Implementation Details:

commit_create: Captures the current index as a tree, reads the current HEAD for the parent pointer, and writes a commit object containing the author, timestamp, and message.
Atomic HEAD Update: Moves the branch reference to the new commit hash only after the commit object is safely on disk.
Phase 4 Verification
📸 Screenshot 4A: <img width="1715" height="917" alt="4a" src="https://github.com/user-attachments/assets/e2655603-44cf-4546-b014-8130dbfb5044" />
Output of ./pes log showing history
Commit History

📸 Screenshot 4B: <img width="1714" height="918" alt="4b" src="https://github.com/user-attachments/assets/f331a5d2-dd44-4e4d-be10-0b5e78a1681c" />
find .pes -type f showing object store growth
Object Store Growth

📸 Screenshot 4C: <img width="2041" height="771" alt="4c" src="https://github.com/user-attachments/assets/57d9360f-65bd-42af-a6ff-21f971289e22" />
cat .pes/HEAD and branch reference verification
HEAD Reference

🏁 Final Integration
All modules (Object, Tree, Index, Commit) were integrated into the main pes binary. The system supports full versioning workflows including initializing repos, staging changes, and viewing persistent history.

📸 Screenshot Final:<img width="1297" height="506" alt="finale1" src="https://github.com/user-attachments/assets/d39fe6ac-651b-4289-8dda-d7c591576637" />
 <img width="1297" height="699" alt="finale2" src="https://github.com/user-attachments/assets/14433ee8-fb44-4198-b78b-e934d7443d91" />
Full integration test output (bash test_sequence.sh)
Integration Tests Integration Tests

🧠 Analysis Questions
Phase 5: Branching and Checkout
Q5.1: Implementation of Checkout To implement pes checkout <branch>, two main steps are required:

Reference Update: The .pes/HEAD file must be updated to point to the new branch (e.g., ref: refs/heads/new-branch).
Working Directory Synchronization: The project files in the working directory must be replaced with the exact snapshots stored in the target branch's tree. This involves walking the target tree, extracting blobs, and writing them to disk while deleting files not present in the new tree. Complexity: It must handle uncommitted changes to prevent data loss.
Q5.2: Dirty Directory Detection We detect a "dirty" state by comparing three sources:

WD vs. Index: Compare the physical file mtime and size with the index entry. Differences indicate "unstaged" changes.
Index vs. HEAD: Compare the index hash with the hash in the current HEAD commit's tree. Differences indicate "staged" but uncommitted changes.
Q5.3: Detached HEAD State In this state, HEAD points directly to a hash. Commits work normally, but they aren't "owned" by any branch. Recovery: Recent hashes can be found in the terminal log or by inspecting .pes/objects for recent commit metadata. A branch can then be created at that hash manually.

Phase 6: Garbage Collection
Q6.1: Mark-and-Sweep Algorithm

Mark: Starting from all branch refs, recursively follow every commit, tree, and blob, adding their hashes to a "reachable" HashSet.
Sweep: Delete any files in .pes/objects/ whose hashes were not marked. Estimation: For 100k commits, you'd visit at least 100k commit objects plus their trees. Deduplication ensures that unique objects are only visited once.
Q6.2: GC Race Conditions If GC runs during a commit, GC might see a new blob that is not yet linked to a commit as "unreachable" and delete it. Git avoids this by using a grace period (keeping objects for a specific number of days regardless of reachability).
