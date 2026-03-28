---
name: shrav
description: "SHRAV (Slack Harvested Recommendations Across Voices) — 社内Slackの全チャンネルから雑談を収集し、ユーザーの興味に合わせてパーソナライズされた話題を毎日届けるスキル。Use when: setting up or running SHRAV, delivering daily/hourly topic recommendations from Slack, handling +1/-1 button feedback from SHRAV messages, running profile maintenance or feedback pruning, or any task mentioning SHRAV, 雑談レコメンド, or Slack雑談収集."
---

# SHRAV — Slack Harvested Recommendations Across Voices

多聞天（ヴァイシュラヴァナ）のように、あらゆるチャンネルの声を聴き、その人に届ける。

## Config (workspace root: `shrav-config.md`)

On first run, if `shrav-config.md` doesn't exist, create it interactively:

```markdown
# SHRAV Config
channels: []            # Channel IDs to monitor. Empty = ask user to populate.
deliver_to: "user:UXXXXXXXX"  # DM target (user:<id> or channel:<id>)
max_topics: 5           # Max recommendations per delivery
max_feedback_entries: 50  # Max rows in shrav-feedback.md before pruning triggers
feedback_retention_days: 30  # Feedback older than this is pruned
last_harvest_ts: null   # Unix timestamp; updated after each harvest
```

Ask the user for `channels` (list of Slack channel IDs to monitor) and `deliver_to` before first harvest. They can add more channels later.

## State Files

| File | Purpose |
|------|---------|
| `shrav-config.md` | Configuration (see above) |
| `shrav-feedback.md` | Feedback log (+1/-1 per post) |
| `shrav-profiling.md` | User interest profile (derived from feedback) |

See `references/state-formats.md` for exact schemas.

## Workflows

### 1. Harvest & Recommend (scheduled or on-demand)

**Trigger:** cron at configured time, or user says "SHRAVを実行して" / "今日の雑談は？"

**Steps:**

1. **Read config** from `shrav-config.md`. Get `last_harvest_ts`, `channels`, `max_topics`.
2. **Collect posts**: For each channel, call `slack readMessages` and keep only messages newer than `last_harvest_ts`. Use `limit: 100` per channel.
3. **Filter casual/non-work**: Use your judgment to filter out messages that are clearly work-related (task assignments, PR reviews, incident alerts, meeting notes, announcements). Keep conversational, personal, curious, playful messages — i.e., *雑談*.
4. **Deduplicate**: Discard any post whose permalink already appears in `shrav-feedback.md` (already shown).
5. **Load profile** from `shrav-profiling.md` (if it exists).
6. **Score & select**: See `references/scoring-guide.md` for the scoring algorithm. Select up to `max_topics` posts.
7. **Post recommendations** as a Slack DM block to `deliver_to`. See `references/blocks-template.md` for the exact block structure with +1/-1 buttons.
8. **Update** `last_harvest_ts` in `shrav-config.md` to current Unix timestamp.

### 2. Feedback Handling (button press)

**Trigger:** User presses +1 or -1 on a SHRAV recommendation block.

Button `action_id` format: `shrav_up_<base64_permalink>` or `shrav_dn_<base64_permalink>`

**Steps:**

1. Decode the permalink from `action_id`.
2. Read `shrav-feedback.md`.
3. If the permalink already has a row, update its `reaction` field.
4. If not, append a new row. See `references/state-formats.md` for schema.
5. Acknowledge the interaction (edit the original message or send an ephemeral "記録しました ✅").
6. If `shrav-feedback.md` row count exceeds `max_feedback_entries`, trigger **Profile Maintenance** (Workflow 3) immediately.

### 3. Profile Maintenance (scheduled cron)

**Trigger:** cron (recommend: once per day, or after Workflow 2 triggers pruning).

**Steps:**

1. Read `shrav-feedback.md` and `shrav-profiling.md`.
2. **Prune old feedback**: Remove rows older than `feedback_retention_days`. If two rows share the same permalink, keep only the most recent.
3. **Re-derive profile**: Analyze the remaining feedback. Group +1 and -1 posts by topic/theme. See `references/scoring-guide.md` for how to write the profile.
4. Write updated `shrav-profiling.md`.
5. Write pruned `shrav-feedback.md`.

## Scheduling

Tell the user to set up two cron jobs:

```
# Harvest & Recommend (e.g., weekday 9am JST = 0am UTC)
"SHRAV harvest and recommendを実行して"  →  cron: 0 0 * * 1-5

# Profile Maintenance (e.g., daily 3am UTC)
"SHRAV profile maintenanceを実行して"   →  cron: 0 3 * * *
```

Use `openclaw cron` to configure, or instruct the user to add these via the OpenClaw UI.

## Slack Interactive Components Note

+1/-1 buttons require the Slack app to have **Interactivity** enabled and a **Request URL** pointing to OpenClaw's Slack webhook endpoint. Confirm with the user that this is configured. If not, describe the setup steps:
1. Go to api.slack.com → Your App → Interactivity & Shortcuts
2. Enable Interactivity
3. Set Request URL to: `https://<your-openclaw-host>/webhooks/slack/interactive`

## Reference Files

- `references/state-formats.md` — Schemas for `shrav-feedback.md` and `shrav-profiling.md`
- `references/blocks-template.md` — Slack Block Kit JSON for recommendations + buttons
- `references/scoring-guide.md` — How to score, rank, and profile topics
