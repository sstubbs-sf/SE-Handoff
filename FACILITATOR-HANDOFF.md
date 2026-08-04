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

## 3. Deploy — paste this one prompt

Fill in the two blanks and send it:

```
Deploy the SnowCup Build-an-Agent Lab into my Snowflake account using the
snowcup-deploy skill.

My snow CLI connection is: <YOUR-CONNECTION-NAME>
Provision these learners: <email@company.com, ANOTHER_USER, ...>

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

## 5. Hand off to learners

The deploy prints these. Learners need **only**:

- The Snowsight sign-in URL
- **The lab app URL** — the training itself, with progress that saves per user
- Their build schema name, e.g. `SNOWCUP_BUILDS.WS_JSMITH`

Tell them: work the modules in order, tick each checkpoint as you go, then submit
answers to the **SnowCup Referee** agent as `Q1: <answer>`. 120 points, 3–4 hours.

> **Do not send learners this package, the SQL, or the guide — they contain the
> answer key.** The lab app URL is the only thing they need.

## 6. During and after the workshop

Open the **admin console** (printed at the end) for the roster, environment
health, challenge leaderboard, and each learner's lab progress. Give another
facilitator access without handing out `ACCOUNTADMIN`:

```sql
GRANT ROLE SNOWCUP_FACILITATOR TO USER <person>;
```

For anything ongoing — adding learners, health checks, teardown — ask Cortex Code
and it will use the companion `snowcup-admin` skill.

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
