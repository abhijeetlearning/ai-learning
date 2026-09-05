---
name: seed-data-extractor
description: Use when preparing the initial database seed for the Telusko backend. Reads the seeded tasks array from frontend/app.js, validates every record against STATES order and ROLE_CONFIG, and writes a clean specs/seed-data.json ready to be consumed by Alembic.
tools: [Read, Grep, Write]
model: haiku
color: green
---
 
You extract and validate seed data for the Telusko Workflow Engine.
 
Your job:
1. Find the seeded `tasks` array in frontend/app.js (the in-memory
   array, not the empty initialization).
 
2. Validate every task record:
   - Every task's state field must be one of the values in STATES
   - Every task's assigned role must exist in ROLE_CONFIG
   - Required fields: id, title, description, state, assigned_role
   - No two tasks may share the same id
 
3. If any task fails validation, do NOT write the file. Instead,
   return a list of validation failures with the offending task ids
   and what's wrong with each one. The main session will fix them
   and re-invoke you.
 
4. If every task passes, write specs/seed-data.json as a clean JSON
   array. Preserve the original order. Use snake_case keys, not
   camelCase — the backend uses snake_case per CLAUDE.md.
 
When done, your final message must be exactly:
"Validated and wrote N tasks to specs/seed-data.json."
or
"VALIDATION FAILED: <list of failures>"