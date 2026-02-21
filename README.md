# Snowflake Skills

Reusable Snowflake investigation skills for debugging, performance analysis,
cost optimization, operations monitoring, governance checks, and security
audits.

Each skill lives in its own folder with a `SKILL.md` file that includes:
- when to use the skill
- required inputs
- SQL templates
- output format and recommendations

For top-level routing guidance, see `../../AGENTS.md`.

## Skill Index

### Performance
- [`optimize-query-by-id`](performance/optimize-query-by-id/SKILL.md)
- [`query-insights`](performance/query-insights/SKILL.md)
- [`table-profiling`](performance/table-profiling/SKILL.md)
- [`clustering-analysis`](performance/clustering-analysis/SKILL.md)

### Cost
- [`find-expensive-queries`](cost/find-expensive-queries/SKILL.md)
- [`credit-consumption-trends`](cost/credit-consumption-trends/SKILL.md)
- [`get-table-sizes`](cost/get-table-sizes/SKILL.md)
- [`warehouse-utilization`](cost/warehouse-utilization/SKILL.md)

### Analysis
- [`data-freshness-check`](analysis/data-freshness-check/SKILL.md)
- [`table-dml-activity`](analysis/table-dml-activity/SKILL.md)
- [`get-lineage`](analysis/get-lineage/SKILL.md)
- [`time-travel`](analysis/time-travel/SKILL.md)

### Operations
- [`monitor-task-failures`](operations/tasks/monitor-task-failures/SKILL.md)
- [`pipeline-run-health`](operations/tasks/pipeline-run-health/SKILL.md)
- [`monitor-long-running-queries`](operations/monitor-long-running-queries/SKILL.md)
- [`monitor-dynamic-table-refreshes`](operations/dynamic-tables/monitor-dynamic-table-refreshes/SKILL.md)
- [`monitor-pipe-failures-and-loads`](operations/pipes/monitor-pipe-failures-and-loads/SKILL.md)
- [`stream-has-data-and-staleness`](operations/streams/stream-has-data-and-staleness/SKILL.md)

### Governance
- [`find-unused-tables`](governance/find-unused-tables/SKILL.md)
- [`find-new-and-dropped-columns`](governance/find-new-and-dropped-columns/SKILL.md)
- [`find-new-tables-and-views`](governance/find-new-tables-and-views/SKILL.md)
- [`find-recent-deletions`](governance/find-recent-deletions/SKILL.md)
- [`most-used-columns`](governance/most-used-columns/SKILL.md)

### Security
- [`audit-login-activity`](security/audit-login-activity/SKILL.md)
- [`audit-role-grants`](security/audit-role-grants/SKILL.md)

## Tool Documentation

- OpenCode skills docs: <https://opencode.ai/docs/skills>
- Codex skills docs (Rules -> Skills): <https://platform.openai.com/docs/codex>
- Cursor rules/context docs: <https://docs.cursor.com/context/rules>

## Contributing

This project is fully open source, and all contributions are welcome.

If you want to contribute:
1. Add or update a skill in `skills/snowflake/<category>/<skill-name>/SKILL.md`.
2. Keep naming/frontmatter consistent (`name`, `description`) and match folder names.
3. Prefer safe, read-first SQL patterns (and MCP-compatible defaults where applicable).
4. Update `../../AGENTS.md` and this README index when adding or renaming skills.
5. Open a pull request with a clear description of the change and rationale.
