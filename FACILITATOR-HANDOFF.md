# SnowCup Build-an-Agent Lab — Facilitator Deployment

You have everything needed to stand up this workshop in a Snowflake account you
control. Deployment is one prompt.

---

## 1. Check you have these

| Requirement | Notes |
|---|---|
| `ACCOUNTADMIN` on the target account | Creates databases, roles, warehouses, and account parameters |
| **A paid account** | Trial accounts cannot run Snowflake App Runtime, and the lab itself is a deployed app — the training cannot be delivered on a trial |
| Cortex Code **Desktop** (not Snowsight) | The app deploy builds a container image through the Snowflake CLI. There is no Snowsight equivalent. |
| Node 20+ and the `snow` CLI | `snow --version` and `node --version` should both answer |
| A working `snow` connection | `snow connection list`. Note the name — you need it for the prompt. |
| **Learner users without `ACCOUNTADMIN`** | A learner who holds `ACCOUNTADMIN` can read the hidden answer key, so the 120-point challenge means nothing for them. The deploy warns you if it detects this. See below. |

### Do not run this on a shared hands-on-lab user

If everyone signs in as the same account user, two things break and neither is
obvious:

- **The answer key is readable.** With `DEFAULT_SECONDARY_ROLES = ALL`, a session
  keeps `ACCOUNTADMIN` active even when the learner's primary role is their
  workshop role — so `SELECT * FROM CHALLENGE_KEY` just works.
- **Progress collapses onto one row.** Checkpoints and scores are keyed on the
  Snowflake user, so a shared user means one shared set of results.

Give each learner their own user with **no** `ACCOUNTADMIN`. The deploy prints a
`CHALLENGE INTEGRITY` warning listing any learner it finds holding it.

Verify the connection reaches the right account before you start:

```bash
snow sql -c <your-connection> --role ACCOUNTADMIN -q "SELECT CURRENT_ACCOUNT_NAME()"
```

## 2. Install the package

Unzip it into your Cortex Code skills directory, then **start a new Cortex Code
session** so the skill loads:

```bash
unzip snowcup-deploy.zip -d ~/.snowflake/cortex/skills/
```

> **Sharing it through git instead?** The unzipped `snowcup-deploy/` folder is
> self-contained and safe to commit as-is — it carries its own `README.md`
> (deployment guide for whoever clones it) and a `.gitignore` that keeps
> `node_modules/` out. **Keep that repository private: the folder contains the
> answer key.**

### Already running an earlier version of this lab? Do this first

Older versions set `DEFAULT_SECONDARY_ROLES = ()` on every learner. The lab no
longer manages that property, and `snowcup-data-setup.sql` now drops the column
that held the original values — so if you deploy straight over an older install,
any learner still sitting at `()` stays there permanently and the record of what
they had is gone. Nothing errors; it is silent.

One read-only script tells you whether this applies to you:

```bash
snow sql -c <your-connection> --role ACCOUNTADMIN -f verify/preflight_secondary_roles.sql
```

- `NOTHING TO MIGRATE` — fresh or already-migrated account. Deploy normally.
- `RESTORE BEFORE DEPLOYING` — it lists each affected learner and prints the exact
  `ALTER USER` to run. Review them, run them, then deploy.

Skip this entirely if you are deploying into an account that has never had the
lab on it.

## 3. Deploy — paste this one prompt

Fill in the two blanks and send it:

```
Deploy the SnowCup Build-an-Agent Lab into my Snowflake account using the
snowcup-deploy skill.

My snow CLI connection is: <YOUR-CONNECTION-NAME>
Provision these learners: <email@company.com, ANOTHER_USER, ...>

If this account already has an earlier version of the lab on it, first run
verify/preflight_secondary_roles.sql and stop to show me the result before
changing anything.

Run the full deployment including both apps. Stop at the first failed gate
rather than continuing. When it finishes, give me the learner hand-off
details and confirm every verification gate passed.
```

Leave the learners line out if you want to add people later — provisioning is
idempotent and can be re-run any time.

## 4. What happens

Roughly 20–40 minutes, mostly waiting on two container builds.

1. Readiness probes (Gen2 warehouses, account edition) — costs nothing
2. App Runtime account defaults
3. Shared tournament data, roles, warehouse
4. Grader, hidden answer key, and the SnowCup Referee agent
5. Learner provisioning — one role, one build schema, and a default role per person. Also repairs ownership: anything in a learner's schema created by another role (a facilitator helping out, or the reference solution built on ACCOUNTADMIN) is handed back to that learner, because otherwise they hold no privilege on it and get no error explaining why
6. **29 verification gates, plus a completion-rule proof** — 8 challenge answers, 6 role/isolation gates, an all-kinds build-schema ownership gate, 14 lab-progress and quiz gates, then a functional test that a *new* completion stamp really does require the checkpoints, the challenge answers, and the reinforcement quizzes together (it runs against a synthetic learner and removes it again)
7. CoWork visibility check
8. The learner lab app and the facilitator admin console
9. All grants, then the hand-off summary

Provisioning runs before verification on purpose: the role-model gates check
per-learner roles and build schemas, so there has to be at least one learner for
them to mean anything. If you deploy without `--learners`, those gates are skipped
with a warning rather than failed — re-run them after provisioning:

```bash
snow sql -c <your-connection> --role ACCOUNTADMIN -f verify/check_roles.sql
```

It stops rather than guessing if the region lacks Gen2 warehouses, or if the
account already has a CoWork object (adding to that is a shared-object decision,
so it reports and leaves it alone).

### What this changes outside the workshop's own objects

Almost nothing, and nothing silently. Everything the deploy creates lives under
`SNOWCUP_*` (plus `SNOWFLAKE_APPS` for the two apps). Specifically:

- **Re-running is safe.** There is no `DROP DATABASE`, no `DROP SCHEMA`, no
  `TRUNCATE`, and no `CREATE OR REPLACE` on any database or schema. Every database
  and schema uses `IF NOT EXISTS`, so re-running never destroys work a learner has
  already built. The deploy is designed to be run repeatedly.
- **One account-level setting may change:** `CORTEX_ENABLED_CROSS_REGION`. Cortex
  orchestration models must be reachable for the Referee agent to answer. It is
  changed **only** when it is currently `DISABLED`, it is never narrowed, and an
  explicit region list is left exactly as it is. When it does change, the deploy
  prints an `ACCOUNT-LEVEL CHANGE` warning naming the old value and the statement
  to restore it. If your organisation set this deliberately, decide before you run.
- **App Runtime account defaults** (`DEFAULT_SNOWFLAKE_APPS_*`) are set **only if
  unset**, and only with `--yes`. An existing value is reported and left alone.
- **Learner `DEFAULT_ROLE` / `DEFAULT_WAREHOUSE`** are set **only if the user has
  none** (unset or `PUBLIC`). An existing default is reported and left alone — the
  output then tells you to have that learner run `USE ROLE SNOWCUP_WS_<NAME>;`.
  Your own defaults are never touched.
- **Nothing is deleted for you.** Teardown statements ship commented out, and the
  only destructive path is the admin console's Deprovision with "also drop their
  build schema" ticked, which additionally requires typing `REMOVE`.

## 5. Checklist — everything you have to account for

Print this. Anything unchecked is a thing that bites you in front of learners.

**Before you deploy**

- [ ] `ACCOUNTADMIN` on the target account
- [ ] Paid account (trials cannot run App Runtime, and the lab *is* an app)
- [ ] `snow --version` and `node --version` both answer
- [ ] `snow connection list` — note the connection name
- [ ] Confirm it points at the right account:
      `snow sql -c <conn> --role ACCOUNTADMIN -q "SELECT CURRENT_ACCOUNT_NAME()"`
- [ ] One Snowflake user per learner, **none holding `ACCOUNTADMIN`**
- [ ] Decide on `CORTEX_ENABLED_CROSS_REGION` if your org set it deliberately (see §4)

**Deploy**

- [ ] Every gate reads `PASS` — the run stops at the first failure, do not continue past one
- [ ] `8/8 answers MATCH` specifically (proves the data and the answer key agree)
- [ ] No `CHALLENGE INTEGRITY` warning, or you accept that those learners can read the key
- [ ] Note any `ACCOUNT-LEVEL CHANGE` warning and its restore statement
- [ ] Both apps report `ready`

**Before learners arrive**

- [ ] Lab app URL captured (§6) and it opens for you
- [ ] Admin console URL captured (§7) and it opens for you
- [ ] Admin console → **Environment health** is all green
- [ ] Roster shows every learner with a role and a build schema
- [ ] Each learner knows their build schema and, if they already had a default role,
      that they must run `USE ROLE SNOWCUP_WS_<NAME>;`
- [ ] Any co-facilitators granted: `GRANT ROLE SNOWCUP_FACILITATOR TO USER <person>;`

**During**

- [ ] Watch **Lab progress** for anyone stalled on one module
- [ ] Watch the same tab's points column for anyone still on 0 once Module 9 starts
- [ ] If a learner reports an empty schema, re-provision them — that repairs ownership

**After**

- [ ] Export the leaderboard / progress if you need a record; nothing is retained for you
      (`SELECT * FROM SNOWCUP_WORKSHOP.GRADING.LEADERBOARD;`)
- [ ] Restore `CORTEX_ENABLED_CROSS_REGION` if you changed it and want it back
- [ ] Deprovision only if you actually want the work gone (§7)

## 6. Hand off to learners

The deploy prints these. Learners need **only**:

- The Snowsight sign-in URL
- **The lab app URL** — the training itself, with progress that saves per user
- Their build schema name, e.g. `SNOWCUP_BUILDS.WS_JSMITH`

Tell them: work the modules in order, tick each checkpoint as you go, then submit
answers to the **SnowCup Referee** agent as `Q1: <answer>`. 120 points, 3–4 hours.

> **Do not send learners this package, the SQL, or the guide — they contain the
> answer key.** The lab app URL is the only thing they need.

### Finding the URLs again after the deploy has scrolled away

The deploy prints both URLs at the end, but you will need them again. Either way
works; the SQL way needs nothing but a worksheet.

```sql
SHOW APPLICATION SERVICES LIKE 'SNOWCUP%' IN SCHEMA SNOWFLAKE_APPS.PUBLIC;
SELECT "name" AS APP, 'https://' || "url" AS URL, "status"
  FROM TABLE(RESULT_SCAN(LAST_QUERY_ID())) ORDER BY 1;
```

```
SNOWCUP_ADMIN_APP   https://ajx...snowflakecomputing.app   RUNNING
SNOWCUP_LAB_APP     https://ijx...snowflakecomputing.app   RUNNING
```

Or from the package directory:

```bash
cd assets/snowcup-lab-node   && snow app open --print-only   # the learner lab
cd assets/snowcup-admin-node && snow app open --print-only   # the admin console
```

`status` should read `RUNNING`. The URLs are stable — they only change if an app is
torn down and redeployed, so they are safe to paste into a calendar invite.

## 7. The admin console

Your own URL, separate from the lab. Opens for `ACCOUNTADMIN` and for anyone holding
`SNOWCUP_FACILITATOR`, so you can delegate without handing out account admin:

```sql
GRANT ROLE SNOWCUP_FACILITATOR TO USER <person>;
```

Five tabs:

| Tab | What it is for |
|---|---|
| **Add learners** | Paste names or emails, comma separated. Either a Snowflake username or an email address works. Idempotent, so re-running is safe — and re-running is the repair path when someone reports an empty schema, because it hands ownership of anything in their build schema back to their own role. |
| **Roster** | Who is provisioned, their role, and their build schema. Check this before learners arrive: a missing row means they cannot start. |
| **Remove** | Offboarding. Revoking the role is the default and keeps their work. Ticking **also drop their build schema** deletes everything they built and additionally requires typing `REMOVE` — it is the only destructive action in the whole console. |
| **Lab progress** | Per learner per training: checkpoints ticked, challenge answers correct, quiz answers correct, challenge points out of the possible 120, and whether they are complete. This is what you watch during the session to spot someone stalled. There is no separate leaderboard tab — for a ranked view query `SNOWCUP_WORKSHOP.GRADING.LEADERBOARD`. Each row also has a **Reset scores** button (see below). |
| **Environment health** | The same invariants the deploy gates check, re-checked live: shared data loaded, answer key intact, every role paired with a schema, trainings registered, and **learners own their own work**. If that last one reports a problem it lists each object and offers **Repair ownership**. |

### Resetting one learner's scores

Each row in **Lab progress** has a **Reset scores** button (two-step: click, then
confirm). Use it when someone wants to re-run the lab, or to clear yourself after a
rehearsal so you do not sit at the top of the leaderboard.

It clears, **for that learner and that training only**:

- every checkpoint they ticked
- every reinforcement quiz answer
- every challenge answer and the points from them
- the completion stamp

It deliberately does **not** touch:

- **anything they built** — their semantic view, agent, Cortex Search service, views
  and procedures are their *work*, not their score. A reset never destroys that.
- their role, build schema, or user defaults. That is offboarding, which is the
  removal section of the **Roster** tab.
- their visit history. First-seen, last-seen and visit count survive, so you can
  still see they had engaged. Note the status column will read `not started`
  regardless, because status is derived from progress rather than visits.

Scoped to one row on purpose: once a second training is registered, a
reset-everything button would wipe work you may not have meant to touch. Re-running
it on an already-clear learner reports "nothing to do" rather than failing.

The equivalent from a worksheet:

```sql
CALL SNOWCUP_WORKSHOP.ADMIN.RESET_LEARNER_PROGRESS('<learner>', 'SNOWCUP');
```

Two things worth knowing about the health tab:
- "Learners own their own work" failing with *could not run `ADMIN.AUDIT_BUILD_OWNERSHIP`*
  means ownership is **unverified, not broken**. Re-run `assets/snowcup-data-setup.sql`
  (idempotent) and reload.
- A wrong owner produces **no error** for the learner — their schema just looks empty,
  or their agent reports that its search service does not exist. That is why this check
  exists, and why re-provisioning fixes it.

For anything ongoing — adding learners, health checks, teardown — ask Cortex Code and
it will use the companion `snowcup-admin` skill.

## If something fails

Send the failure back to Cortex Code in the same session; the skill has a
troubleshooting table covering the common causes. The most frequent by far:
`snow app deploy` run **without** `--role ACCOUNTADMIN` fails on the access
integration, and an app deployed without account defaults lands in a personal
database where it cannot be granted to anyone.

Prefer to skip the agent and run it yourself? The same deployment is one command:

```bash
cd ~/.snowflake/cortex/skills/snowcup-deploy
./deploy.sh --connection <your-connection> --yes --learners "a@x.com, BJONES"
```
