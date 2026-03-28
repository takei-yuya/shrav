# SHRAV Slack Block Template

## Recommendation Message Structure

Send via `slack sendMessage` with `blocks` payload. Replace `{{...}}` placeholders.

```json
{
  "blocks": [
    {
      "type": "header",
      "text": {
        "type": "plain_text",
        "text": "📡 今日の SHRAV レコメンド",
        "emoji": true
      }
    },
    {
      "type": "context",
      "elements": [
        {
          "type": "mrkdwn",
          "text": "{{DATE}} | チャンネル横断で見つけた気になる雑談、最大{{MAX_TOPICS}}件"
        }
      ]
    },
    {
      "type": "divider"
    },

    // ---- Repeat this block group for each recommended post ----
    {
      "type": "section",
      "text": {
        "type": "mrkdwn",
        "text": "*<{{PERMALINK}}|#{{CHANNEL_NAME}} — {{AUTHOR_NAME}}>*\n{{TEXT_PREVIEW}}"
      }
    },
    {
      "type": "actions",
      "elements": [
        {
          "type": "button",
          "text": {
            "type": "plain_text",
            "text": "👍 気になる",
            "emoji": true
          },
          "style": "primary",
          "action_id": "shrav_up_{{PERMALINK_B64}}",
          "value": "{{PERMALINK}}"
        },
        {
          "type": "button",
          "text": {
            "type": "plain_text",
            "text": "👎 興味なし",
            "emoji": true
          },
          "style": "danger",
          "action_id": "shrav_dn_{{PERMALINK_B64}}",
          "value": "{{PERMALINK}}"
        }
      ]
    },
    {
      "type": "divider"
    }
    // ---- End repeat ----
  ]
}
```

## Placeholder Reference

| Placeholder | Value |
|-------------|-------|
| `{{DATE}}` | e.g., `2024-03-11（月）` |
| `{{MAX_TOPICS}}` | from config |
| `{{PERMALINK}}` | full Slack URL |
| `{{CHANNEL_NAME}}` | e.g., `random` (without `#`) |
| `{{AUTHOR_NAME}}` | display name from `memberInfo` |
| `{{TEXT_PREVIEW}}` | first 100 chars, truncate with `…` if longer |
| `{{PERMALINK_B64}}` | URL-safe base64 of the permalink (for action_id; keep short — use only the path segment `archives/CXXX/pTIMESTAMP` if full URL exceeds Slack's 255-char action_id limit) |

## PERMALINK_B64 Encoding

```python
import base64
path = permalink.split("slack.com")[1]  # e.g., /archives/C123/p1710000000000000
b64 = base64.urlsafe_b64encode(path.encode()).decode().rstrip("=")
```

Decode on feedback receipt:
```python
padded = b64 + "=" * (-len(b64) % 4)
path = base64.urlsafe_b64decode(padded).decode()
permalink = f"https://yourteam.slack.com{path}"
```

## No-results Message

If no casual posts were found since last harvest:

```json
{
  "blocks": [
    {
      "type": "section",
      "text": {
        "type": "mrkdwn",
        "text": "📡 *SHRAV*: 前回の実行以降、おすすめできる雑談が見つかりませんでした。モニター対象チャンネルを増やすか、次回をお待ちください 🌿"
      }
    }
  ]
}
```

## Feedback Acknowledgement (ephemeral or message edit)

After a button press, reply with an ephemeral message or edit the message:

- +1: `✅ 記録しました！似たような話題を積極的にピックアップします。`
- -1: `✅ 記録しました！この系統の話題は控えるようにします。`
