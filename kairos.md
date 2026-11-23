---
layout: default
title: Two bugs in Kairos, and the one idea behind both
---

# Two bugs in Kairos, and the one idea behind both

Kairos is an open source fraud and risk decision engine. You send it events. It answers approve, step up, review or block. Held cases go to a person.

On November 4 I ran a readfirst review on a fork of Kairos at commit e8fb857. It found the two bugs below. On November 7 the founder asked for a write up. I sent his team ten reproduced bugs, these two among them. They fixed all ten that same day (dcc5057, 90ab622).

Both are easy mistakes to make, and most codebases have some like them. Each is a rule the code kept in one place and lost in another.

## 1. Two doors, one lock

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

So a member, the lowest role, got a 403 from the first route. The same request to the second route got a 200. The mode was now off. The Kairos docs leave environment settings to admins. Another route already had the right check. This one did not.

The fix is two lines.

```ts
if (c.get("userId") && !atLeast(c.get("role"), "admin"))
  return error(c, 403, "forbidden", "Changing client settings needs the admin role, or a secret key");
```

Each route looks fine on its own. The bug only shows when you ask which routes reach clients.save and what each one checks.

## 2. "Is this live?" answered by the clock

A live review or block should open a case and start any workflow. A backtest should do neither.

Kairos told the two apart by the time gap.

```ts
// A decision as of a past moment is a test of the past: it starts no workflow and makes no case
const live = now.getTime() >= savedAt.getTime() - 60_000;
```

The automatic job passes the event's own time. So a real decision saved more than 60 seconds after its event counted as a backtest. That happens when the worker runs behind. It happens when the model fails once and answers on a retry. It happens when someone sends an older event.

The decision was saved and the webhook went out. But no case opened and no workflow started. The person was held and nobody was asked to look. Nothing failed, so nothing showed it.

The fix stops guessing. The caller now says whether the decision is live.

```ts
const { live } = flow;
```

## The idea behind both

Both bugs are a rule that held where it was written and nowhere else.

Only admins change client settings. That held on one route of two.
A backtest opens no case. That held through a guess that also caught slow live decisions.

Neither shows up in a test of one route or one function. That is why both sat in a codebase with tests. You see them when you write the rule down and check every path that reaches it.

A readfirst review found these two. readfirst also keeps rules like these in a map next to the code. Whoever changes that code reads them first.

## Timeline

November 4. A readfirst review of a fork at e8fb857 finds both bugs.
November 7. The founder asks for a write up. Ten reproduced bugs go to his team.
November 7. All ten are fixed on main.
November 13. The draft goes to the founder.

Thanks to the Kairos team for fixing them so fast.
