# claude-code-sandbox-with-skills

MCP server that spins up **Claude Code sandboxes with a bundled Skills pack** on E2B and runs jobs inside them.

You call the MCP once with “here’s a repo + a job”.  
It provisions a fresh sandbox image with **Claude Code preinstalled**, injects this repo’s **Skills bundle** into that sandbox, and lets Claude Code run the job end-to-end inside that environment.

No local `.claude/skills` setup, no manual Skills install in your project.

---

## What this actually gives you

- **Claude Code image baked into the sandbox**
  - The sandbox comes up with Claude Code already available.
  - You’re not installing tools at runtime; you’re starting from a prebuilt “Claude Code worker” image.

- **Bundled Skills pack shipped into the sandbox**
  - This repo contains a set of Skills (as directories with `SKILL.md` + supporting files) designed for repo work.
  - On sandbox startup, the MCP server syncs these Skills into the sandbox filesystem next to Claude Code.
  - Claude Code inside the sandbox sees and uses *this* Skills pack, not whatever happens to be in the user’s repo.

- **Job-level entrypoints instead of low-level commands**
  - From the client side, you don’t micromanage tests or patches.
  - You submit higher-level jobs like:
    - “bootstrap this repo”
    - “run and fix tests”
    - “refactor this component and verify”
    - “run this data experiment”
  - The sandbox-side Claude Code + Skills decide how to execute the job.

- **E2B as the execution substrate**
  - All of the above runs in E2B sandboxes:
    - sandbox lifecycle,
    - resource isolation,
    - long-running jobs.
  - You never talk to E2B directly; the MCP server owns that contract.

---

## High-level flow

1. **Client → MCP:**  
   Send a job to this server (repo pointer + job description / job type).

2. **MCP → E2B:**  
   - Provision an E2B sandbox from a base image that already contains Claude Code.
   - Copy the bundled Skills set from this repo into the sandbox.
   - Copy or mount the target repo into the sandbox (depending on your configuration).

3. **Inside sandbox:**  
   - Start Claude Code in “worker” mode pointed at the repo.
   - Instruct it (via the Skills bundle) to execute the requested job:
     - pick the right Skill(s),
     - follow SKILL.md instructions,
     - call any needed tools / commands.

4. **Sandbox → MCP:**  
   - Stream back logs and status,
   - summarize results (tests, diffs, artifacts, summaries),
   - mark the job as finished.

5. **MCP → client:**  
   - Expose job state and final result over a compact interface (job id, status, result summary, optional artifacts).

From the outside, you only deal with **jobs**.  
The Claude Code + Skills orchestration and the E2B plumbing are entirely encapsulated.

---

## Core concepts

- **Claude-Code-first sandboxes**
  - The “unit” of work is a sandbox that already knows how to:
    - open a repo,
    - read Skills,
    - run typical dev flows (tests, edits, scripts).
  - You don’t ask for “run this shell command”; you ask for “run this job in a Claude Code sandbox”.

- **Bundled Skills**
  - Skills in this repo are treated as a **sealed bundle**:
    - versioned with this MCP server,
    - copied into the sandbox as-is,
    - not dependent on the target repo’s own `.claude/skills`.
  - Updating behavior = updating this repo + redeploying the MCP.

- **Job abstraction**
  - Everything is framed as “job requests” and “job results”.
  - Jobs are long-lived, can stream intermediate output, and then resolve into a compact result object (e.g., test summary, diff summary, metrics).

- **No local setup**
  - Users do **not** need:
    - to install Claude Code locally,
    - to manage `.claude/skills` in every project,
    - to wire E2B credentials into their IDE.
  - They only need to be able to call this MCP server.

---

## Intended usage

- When you want **remote Claude Code workers** that:
  - come with a **fixed, vetted Skills bundle**,
  - can safely operate on repos,
  - can be spun up and torn down per job.

- When you don’t want:
  - to ship Skills into each repo,
  - to expose raw E2B / sandbox APIs to the LLM,
  - to manage Claude Code installs per developer machine.

This project is the “**Claude Code + Skills on E2B, behind a single MCP endpoint**” building block you can wire into your editor, CI, or higher-level agent system.
