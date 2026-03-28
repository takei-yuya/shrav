# SHRAV State File Formats

## shrav-feedback.md

Markdown table. One row per shown post. Updated by Workflow 2 (feedback) and pruned by Workflow 3 (maintenance).

```markdown
# SHRAV Feedback

| id | permalink | channel_id | ts | shown_at | reaction | topic_tags |
|----|-----------|------------|----|----------|----------|------------|
| 1  | https://yourteam.slack.com/archives/C123/p1710000000000000 | C123 | 1710000000.000000 | 2024-03-10T09:00:00Z | +1 | 雑談,旅行,食べ物 |
| 2  | https://yourteam.slack.com/archives/C456/p1710100000000000 | C456 | 1710100000.000000 | 2024-03-10T09:00:00Z | -1 | 技術,Python |
| 3  | https://yourteam.slack.com/archives/C789/p1710200000000000 | C789 | 1710200000.000000 | 2024-03-10T09:00:00Z |    | ゲーム,音楽 |
```

**Columns:**
- `id` — Auto-increment integer
- `permalink` — Full Slack permalink (used as unique key)
- `channel_id` — Channel where the post was found
- `ts` — Slack message timestamp (float string)
- `shown_at` — ISO 8601 UTC timestamp when SHRAV recommended this
- `reaction` — `+1`, `-1`, or empty (no reaction yet)
- `topic_tags` — Comma-separated inferred tags (assigned at harvest time, used for profiling)

**Deduplication rule:** If the same permalink appears twice, keep only the row with the most recent `shown_at`. Always prefer the row that has a reaction over one that doesn't.

---

## shrav-profiling.md

Free-form Markdown profile. Maintained by Workflow 3. Used by Workflow 1 (scoring).

```markdown
# SHRAV User Interest Profile

_Last updated: 2024-03-11T03:00:00Z_
_Based on N feedback entries (M positive, K negative)_

## Interests (show more like these)

- **旅行・グルメ**: 海外旅行、ローカルフード、レストラン開拓に強い反応。特に東南アジア系の話題でhigh。
- **音楽・ライブ**: コンサート情報や「聴いてるもの」共有に+1が集中。
- **ペット**: 猫・犬の投稿に好反応。

## Dislikes (suppress these)

- **技術詳細の議論**: コードレビューや技術的なツール話題に-1が多い（仕事感が強いためか）。
- **スポーツ**: 野球・サッカー関連の投稿に-1傾向。

## Neutral / Serendipity candidates

- **ゲーム**: まだ反応データが少ない。新たな興味発見の可能性あり。
- **映画・ドラマ**: 反応が混在。好きなジャンル依存と推定。

## Profile Notes

フードや旅行の個人的な体験談が特に好まれる。情報共有よりも「ちょっと聞いてよ」系の雑談に強く反応する傾向。
```

**Writing guidelines:**
- Keep it concise but specific — cite actual topics, not just "likes casual talk"
- Update the `Last updated` and count lines each run
- The Interests / Dislikes / Neutral sections are the primary signal for Workflow 1 scoring
- Add Profile Notes for nuance that doesn't fit the categories
