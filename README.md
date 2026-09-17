# ainoflow-skills

Public Agent Skills that show how real business roles share structured state across agents, apps, and users with [Ainoflow MCP](https://www.ainoflow.io/docs/mcp).

Agents that authenticate with the same Ainoflow API key see the same Memory documents, Storage JSON, Files, and Inbox. These skills teach an agreed `category`/`key` namespace so an intake agent can write a record and a reviewer agent — in another chat, tool, or app — can read it without copying files around.

This repo is a lead magnet and teaching library. It uses only the public product surface: [Docs](https://www.ainoflow.io/docs) and [MCP](https://www.ainoflow.io/docs/mcp). It does not document private APIs.

## Why Ainoflow for multi-agent state

Use each MCP service for what it is built to do:

| Service | MCP endpoint | Role in a handoff |
| --- | --- | --- |
| [Memory](https://www.ainoflow.io/docs/mcp/memory) | `https://mcp.ainoflow.io/mcp/v1/memory` | Markdown notes with revisions, `[[wiki-links]]`, and ranked recall. Find knowledge without knowing the exact key. |
| [Storage](https://www.ainoflow.io/docs/mcp/storage) | `https://mcp.ainoflow.io/mcp/v1/storage/json` | Schema-free JSON workflow state by `category`/`key`. Optional TTL. Partial updates via JSON Merge Patch (RFC 7386). |
| [Files](https://www.ainoflow.io/docs/mcp/files) | `https://mcp.ainoflow.io/mcp/v1/files` | Binary originals (PDFs, receipts, contracts) addressed by `category`/`key`. |
| [Inbox](https://www.ainoflow.io/docs/mcp/inbox) | `https://mcp.ainoflow.io/mcp/v1/inbox` | List and handle inbound email, then fetch attachment URLs. |

The MCP host is **`mcp.ainoflow.io`**. It is not `api.ainoflow.io` — that host serves REST and does not route `/mcp/v1/*`.

Shared state works because:

1. The same Bearer API key (and therefore the same Ainoflow scope/project) is configured on every agent that should collaborate.
2. Skills in this repo agree on category names (`invoices`, `expenses`, `inbox-triage`, `research`, `vendors`) and path-like keys.
3. Agent A writes Storage + Files + Memory. Agent B lists or searches those same addresses. No private bus is required.

Call each surface's `*_guide` tool first. Public docs and the live guide describe the same contract; if they ever disagree, the guide is current.

## Connect MCP

Replace `YOUR_API_KEY` with a key from your Ainoflow account. **Never commit the real key.**

The snippets below follow the Cursor / Claude pattern published on [MCP](https://www.ainoflow.io/docs/mcp): `mcpServers` entries with `url` and an `Authorization: Bearer` header.

### Cursor

Add the four servers to your Cursor MCP config:

```json
{
  "mcpServers": {
    "ainoflow-memory": {
      "url": "https://mcp.ainoflow.io/mcp/v1/memory",
      "headers": {
        "Authorization": "Bearer YOUR_API_KEY"
      }
    },
    "ainoflow-storage": {
      "url": "https://mcp.ainoflow.io/mcp/v1/storage/json",
      "headers": {
        "Authorization": "Bearer YOUR_API_KEY"
      }
    },
    "ainoflow-files": {
      "url": "https://mcp.ainoflow.io/mcp/v1/files",
      "headers": {
        "Authorization": "Bearer YOUR_API_KEY"
      }
    },
    "ainoflow-inbox": {
      "url": "https://mcp.ainoflow.io/mcp/v1/inbox",
      "headers": {
        "Authorization": "Bearer YOUR_API_KEY"
      }
    }
  }
}
```

### Claude Desktop

The same `mcpServers` object works in Claude Desktop MCP settings:

```json
{
  "mcpServers": {
    "ainoflow-memory": {
      "url": "https://mcp.ainoflow.io/mcp/v1/memory",
      "headers": {
        "Authorization": "Bearer YOUR_API_KEY"
      }
    },
    "ainoflow-storage": {
      "url": "https://mcp.ainoflow.io/mcp/v1/storage/json",
      "headers": {
        "Authorization": "Bearer YOUR_API_KEY"
      }
    },
    "ainoflow-files": {
      "url": "https://mcp.ainoflow.io/mcp/v1/files",
      "headers": {
        "Authorization": "Bearer YOUR_API_KEY"
      }
    },
    "ainoflow-inbox": {
      "url": "https://mcp.ainoflow.io/mcp/v1/inbox",
      "headers": {
        "Authorization": "Bearer YOUR_API_KEY"
      }
    }
  }
}
```

These are the **category-agnostic** endpoints (`category` is an argument on each tool). Public docs also describe category-bound URLs such as `/mcp/v1/memory/{category}` if you want one server per namespace.

## Install and use these skills

Each folder under [`skills/`](skills/) is an [Agent Skill](https://cursor.com/docs/skills): a directory that contains `SKILL.md` with YAML frontmatter (`name`, `description`) and the workflow body.

Cursor discovers skills from (see [Cursor Agent Skills](https://cursor.com/docs/skills)):

| Location | Scope |
| --- | --- |
| `.cursor/skills/` or `.agents/skills/` | This project |
| `~/.cursor/skills/` or `~/.agents/skills/` | Your user account, every workspace |
| `.claude/skills/` or `~/.claude/skills/` | Claude-compatible clients |

**Install (pick one):**

1. Copy the skill folders you want into a discovery path, keeping `SKILL.md` as a direct child of the skill folder:
   - Project: `skills/invoice-intake` → `.cursor/skills/invoice-intake/SKILL.md`
   - Personal: `skills/invoice-intake` → `~/.cursor/skills/invoice-intake/SKILL.md`
2. Or clone this repo and copy `skills/*` into `.cursor/skills/` (Cursor) or `~/.claude/skills/` (Claude).
3. In Cursor Agent chat, type `/` and choose a skill by name, or describe the job (invoice intake, expense handoff, inbox triage, research notes, vendor files) and let the agent match the `description`.

Do not nest an extra directory from a zip extract. `SKILL.md` must sit immediately inside the skill folder, and the frontmatter `name` must match that folder name.

## Try it (5 minutes)

Prove the handoff with **one** skill and the four public MCP servers. No Inbox hook is required.

1. **Connect the four MCP servers** from [Connect MCP](#connect-mcp) (Memory, Storage, Files, Inbox) with `YOUR_API_KEY`. Reload MCP so the `*_guide` tools appear.
2. **Install one skill:** copy [`skills/research-notes-memory/SKILL.md`](skills/research-notes-memory/SKILL.md) to `.cursor/skills/research-notes-memory/SKILL.md` (or `~/.cursor/skills/research-notes-memory/SKILL.md`).
3. **Run the sample prompt:** paste [`prompts/research-notes-memory.md`](prompts/research-notes-memory.md) into an agent chat that has those MCP servers. Type `/research-notes-memory` if your host lists skills that way.
4. **Success looks like** these addresses existing under the **same** API key:
   - Memory category `research`, keys `briefs/smoke-q3-pricing`, `notes/smoke-q3-pricing/packaging`, `outlines/smoke-q3-pricing`
   - Storage category `research`, key `sessions/smoke-q3-pricing` with `"status": "ready_for_draft"` and `"readyForDraft": true`

Confirm with `memory_read` on the brief, `memory_search` (`q`: `packaging`, `prefix`: `notes/smoke-q3-pricing/`), and `storage_json_get` on the session. Then delete only those smoke keys (see [TESTING.md](TESTING.md#cleanup)).

Copy-paste prompts for every skill live in [`prompts/`](prompts/). Full smoke A/B/C steps, pass/fail, and cleanup: [TESTING.md](TESTING.md).

## Skill catalog

| Skill | Folder | When to use | Surfaces |
| --- | --- | --- | --- |
| Invoice intake | [`skills/invoice-intake`](skills/invoice-intake/SKILL.md) | Vendor invoices arrive by email or PDF and must be queued for AP review | Inbox, Files, Storage, Memory |
| Expense tracking handoff | [`skills/expense-tracking-handoff`](skills/expense-tracking-handoff/SKILL.md) | An employee submits a report and receipts; finance reviews the same records | Storage, Files, Memory |
| Inbox triage to Storage | [`skills/inbox-triage-to-storage`](skills/inbox-triage-to-storage/SKILL.md) | Route inbound mail into support / billing / sales queues for specialist agents | Inbox, Storage, Memory |
| Research notes in Memory | [`skills/research-notes-memory`](skills/research-notes-memory/SKILL.md) | Researcher and writer share wiki-linked notes plus session progress | Memory, Storage |
| Vendor onboarding files | [`skills/vendor-onboarding-files`](skills/vendor-onboarding-files/SKILL.md) | Collect contracts and onboarding packets, then hand a checklist to compliance | Files, Storage, Memory |

## Public docs

- [Ainoflow docs](https://www.ainoflow.io/docs)
- [MCP overview](https://www.ainoflow.io/docs/mcp)
- [Memory](https://www.ainoflow.io/docs/mcp/memory) · [Storage](https://www.ainoflow.io/docs/mcp/storage) · [Files](https://www.ainoflow.io/docs/mcp/files) · [Inbox](https://www.ainoflow.io/docs/mcp/inbox)
- This repo: [TESTING.md](TESTING.md) · [`prompts/`](prompts/)

Published defaults (confirm with each `*_guide` for your key): Memory documents up to 100 KB; Storage JSON up to 10 MB; Files decoded size is reported as `filesFileSizeBytes` (docs illustrate a 100 MB default). Inbox tools list and handle mail; they do not create inboxes.

## Security

- Never commit API keys, `.env` files, or real inbox hook addresses. Copy [`.env.example`](.env.example) locally (`AINOFLOW_API_KEY=YOUR_API_KEY`).
- Use the `YOUR_API_KEY` placeholder in any snippet you paste into this repo or a ticket.
- Treat Memory, Storage, Files, and Inbox as shared project data. Anyone with the same key can read what these skills write.
- Prefer agreed namespaces over dumping everything into the Memory default category `main`.

## License

[MIT](LICENSE)
