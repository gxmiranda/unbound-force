---
tag: dewey
author: yvonne-devlin
category: gotcha
created_at: 2026-06-23T12:31:08Z
identity: dewey-20260623T123108-yvonne-devlin
tier: draft
---

The Dewey MCP server configuration in opencode.json was set to "type": "local" which spawns a new dewey serve process on each OpenCode session. In a devcontainer environment where startup.sh already starts dewey serve --http :3333 (which acquires an exclusive SQLite lock on graph.db), the OpenCode-spawned instances cannot access the persistent store and fall back to in-memory mode (persistent: false, 0 pages). The fix is to change opencode.json to "type": "remote" with url "http://localhost:3333/mcp/" to connect to the already-running HTTP server. However, since opencode.json is committed to the repo and affects all contributors, the fix should be applied locally with `git update-index --assume-unchanged` to avoid impacting the remote repo.
