# Workshop 4 - Reset Utility

*Optional SQL cleanup utility for workshop reruns.*

---

## What this is

`cleartables.sql` drops every table and view in the `bronze`, `silver`, and
`gold` schemas, so you can rerun Workshop 2 and Workshop 3 from a clean
database without recreating the schemas themselves (`bronze`, `silver`,
`gold` stay in place - only their contents are removed).

It's driven off `sys.tables`/`sys.views` rather than a hardcoded table
list, so it also drops any table or view a pipeline YAML added later - no
edits needed here when Workshop 2's Gold schema grows.

> [!WARNING]
> This is destructive and irreversible. It drops views before tables
> (Gold -> Silver -> Bronze order, since Silver/Gold views would otherwise
> read from tables that no longer exist) across all three schemas in one
> execution, with no confirmation prompt. Do not run this against a
> database you aren't ready to fully rebuild.

## How to use it

1. Navigate to the `workshop4` folder.
2. Open the VS Code SQL Server extension connection for your database
   (same `sqlconection` value as the other workshop folders' `.env`).
3. Open `cleartables.sql` and run it with the SQL Server extension's run
   button.
4. To rebuild, start again from
   [Workshop 2 - Part 1](../workshop2/workshop2-part1-bronze-erp-crm.md).

---

**Back to:** [Workshop overview](../readme.md)
