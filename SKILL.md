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
3. **Filter: 雑談のみ残す（仕事は絶対に除外）**: SHRAVのコンセプトは「雑談でコミュニケーションを活性化すること」。以下の基準で厳格にフィルタリングする。

   **✅ 雑談として残すもの（すべて該当する必要はない）:**
   - 個人的な体験・気づき・近況報告（「週末〇〇行ってきた」「最近〇〇にハマってる」）
   - 趣味・娯楽・食・旅行・ペット・音楽・映画・ゲームなどの話題
   - 世間話・ジョーク・面白いリンク・雑感
   - 「ちょっと聞いてよ」系の投稿
   - 絵文字だけ・スタンプ的な反応でも文脈が雑談なら可

   **❌ 仕事として除外するもの（1つでも該当したら除外）:**
   - タスク依頼・進捗報告・締め切り・KPI・予算・数値目標
   - PRレビュー・バグ報告・インシデント・デプロイ・リリース
   - 会議の設定・議事録・MTG summary
   - 採用・人事・評価・組織変更
   - ツール導入・インフラ・セキュリティの話
   - お知らせ・アナウンス・全社連絡
   - 「〇〇さん、確認お願いします」のような依頼形式

   **グレーゾーンは「仕事寄りか？」で判断: 少しでも仕事の匂いがするなら除外する。**
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
