---
title: AIOZ Jira site, projects, and member roster
type: reference
updated: 2026-08-21
---

# AIOZ Jira site, projects, and member roster

Atlassian site: `https://aioz-team-t7dcm9pq.atlassian.net`
cloudId: `8522fc0d-9027-4f42-a5af-42de8ff63c9e`
MCP scope granted: `read:jira-work` only (read-only; no create/transition).

## Projects (7)

| Key | Name | Style |
|---|---|---|
| AGM | AIOZ Global Map | next-gen |
| ALP | AIOZ Landing page | classic |
| AP | AIOZ Pin | classic |
| AS | AIOZ Storage | next-gen |
| AST | AIOZ Stream | classic |
| DP | AIOZ DePIN | next-gen |
| FETEAM | FETeam | next-gen |

## Human accounts (14 of 28 total; other 14 are apps/bots)

| Display name | accountId | Active projects |
|---|---|---|
| Tuan Tran (me, tuan.quang.tran@aioz.io) | 712020:d5663bcf-359b-4f4c-88c8-9e00ee3387c3 | AGM |
| anh.ngoc.nguyen | 712020:b38cb401-6456-4ee9-bbd2-fe8905c05977 | AGM |
| dat.nguyen | 712020:a046a8c5-e339-479f-88c2-0d474caaba16 | ALP, AST, AGM |
| nam.quoc.le | 712020:445053f0-cc74-40d2-87e8-99faa510609a | AGM, AST |
| hoang.huy.nguyen | 712020:700c80a0-9aea-418d-bafe-659eb63cee5f | AGM |
| thuan.van.pham | 712020:8b3dee9e-ad75-4aa2-94fe-7c6ee5031bf4 | AST |
| khoi.quang.le | 712020:7a02b460-d38f-40c6-80bc-7ff8653f6ecd | AGM |
| anh.quynh.vu | 712020:201f0164-d8d1-4780-a6c4-5a56c09982b9 | AST |
| tuyen.hon.pham | 712020:4fa78969-4285-4d79-9670-81975d196067 | AGM |
| Quang Le | 70121:5d678af2-692b-4f7f-b8b7-1209de020ed2 | DP |
| dat.van.pham | 712020:71a31167-3113-4bb5-bde7-b01be3110565 | AS |
| ngan.thi.vo | 712020:23f73f8a-e335-47aa-8b08-84eea279562a | AGM |
| Anh Dang | 712020:4da45b8b-af1e-4874-a92a-cbf1548af681 | AGM |
| Tue Phan | 63b792c6741248746bf8ac0c | (no recent assignments) |

## How to enumerate users with this MCP

There is no list-users tool. `lookupJiraAccountId` matches a **word prefix** of
displayName or email (space-separated words only - dots are NOT separators), and
returns at most 5 users per call plus a `total`. Enumerate by querying each letter
a-z, then refining any letter whose `total` > 5 with two-letter prefixes.
`searchString: "@"` returns the site-wide total (28 here).

Related: [[jira-task-workflow-standards]]
