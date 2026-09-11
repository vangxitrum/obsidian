---
title: My Jira team (Tuan's team)
type: project
updated: 2026-08-21
---

# My Jira team

Confirmed by Tuan on 2026-08-21 when asked to pick his team from the site roster.
This is *the* team to assume whenever Tuan says "my team", "the team", "my members",
or asks for team workload / standup / sprint status without naming anyone.

Site: `aioz-team-t7dcm9pq.atlassian.net`, cloudId `8522fc0d-9027-4f42-a5af-42de8ff63c9e`.

| Role | Member | accountId |
|---|---|---|
| **Lead** | Tuan Tran (tuan.quang.tran@aioz.io) | 712020:d5663bcf-359b-4f4c-88c8-9e00ee3387c3 |
| Member | nam.quoc.le | 712020:445053f0-cc74-40d2-87e8-99faa510609a |
| Member | khoi.quang.le | 712020:7a02b460-d38f-40c6-80bc-7ff8653f6ecd |
| Member | dat.van.pham | 712020:71a31167-3113-4bb5-bde7-b01be3110565 |
| Member | Anh Dang | 712020:4da45b8b-af1e-4874-a92a-cbf1548af681 |
| Member | ngan.thi.vo | 712020:23f73f8a-e335-47aa-8b08-84eea279562a |

Tuan is the **lead**, so lead-side duties from [[jira-task-workflow-standards]] apply
to him (grooming, Definition of Ready checks, due-date risk reporting), not member duties.

## JQL snippet for the team

```
assignee IN (
  "712020:d5663bcf-359b-4f4c-88c8-9e00ee3387c3",
  "712020:445053f0-cc74-40d2-87e8-99faa510609a",
  "712020:7a02b460-d38f-40c6-80bc-7ff8653f6ecd",
  "712020:71a31167-3113-4bb5-bde7-b01be3110565",
  "712020:4da45b8b-af1e-4874-a92a-cbf1548af681",
  "712020:23f73f8a-e335-47aa-8b08-84eea279562a"
)
```

Members are spread across projects: nam.quoc.le, khoi.quang.le, Anh Dang and
ngan.thi.vo work mostly in AGM; dat.van.pham in AS. Do NOT filter the team by a
single project key - filter by assignee.

Full site roster and the other 8 human accounts: [[jira-site-and-roster]]
