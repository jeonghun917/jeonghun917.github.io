---
layout: default
title: Two bugs in Kairos, and the one idea behind both
---

# Two bugs in Kairos, and the one idea behind both

Kairos is an open source fraud and risk decision engine.

On November 4 I ran a readfirst review(my own dev tool) and I found ten bugs on it and reproduced it.

On November 7 the founder asked for a full report. I sent it to his team. Even if I suggested to them my patch, surprisingly they fixed all of them same day.

two of them deserve to be known widely, I wrote this bug report.

## 1. Two doors, but one lock

Client settings decide what Kairos does with live traffic. One of them is the enforcement mode. Set it to off and fraud checks stop.

Two routes could write these settings.

PUT /app/workspaces/:workspace/client-settings is the one the console uses. It checked for the admin role.

PUT /v1/client-settings is the API route. It also took a signed in session. It checked no role.

```ts
// /app: admins only
app.put(path, async (c) => {
  const a = await access(c, "admin");
  ...
  const saved = await clients.save(a.workspace, a.environment, body.data);

// /v1: no role check
app.put("/client-settings", async (c) => {
  const body = await parse(c, clientSettingsBody);
  ...
  const saved = await clients.save(c.get("workspaceId"), c.get("environmentId"), body.data);
```

So the lowest role could have permissions that is theoretically belongs to owner just by passing the second route.

My suggestion to fix it was like the below and it is received.

```ts
if (c.get("userId") && !atLeast(c.get("role"), "admin"))
  return error(c, 403, "forbidden", "Changing client settings needs the admin role, or a secret key");
```

Each route looks fine on its own. The bug only shows when you ask which routes reach clients.save and what each one checks.

## 2. "Is this live?" answering and answered by the clock

A live review or block should open a case and start any workflow. A backtest should do neither.

Kairos told the two apart by the time gap.

```ts
// A decision as of a past moment is a test of the past: it starts no workflow and makes no case
const live = now.getTime() >= savedAt.getTime() - 60_000;
```

The automatic job passes the event's own time. When it doesn't match the time then, it isn't considered as live. But when it comes to the worker running behind, it is saved more than 60 seconds and user could be behind with nothing failed on logs, while nobody realizes someone needs help.

The fix I suggested was simple, just stopping guessing. The caller now says whether the decision is live

```ts
const { live } = flow;
```

## The common problem behind both

Both bugs are a rule that held where it was written and nowhere else.

Only admins change client settings. That held on one route of two.

A backtest opens no case. That held through a guess that also caught slow live decisions.

Nothing shows up in a test of one route or one function. That is why both sat in a codebase with tests. You see them when you write the rule down and check every path that reaches it.

A readfirst review found these two. readfirst also keeps rules like these in a map next to the code. Whoever changes that code reads them first.
