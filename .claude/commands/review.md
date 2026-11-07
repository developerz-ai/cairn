---
description: Review the current branch with the reviewer (and security auditor when relevant).
---
Run the `reviewer` subagent on `git diff main...`. If the diff touches `src/object`, `src/net`, auth, ACL or path code, also run `security-auditor`. Merge both into one findings table, highest severity first.
