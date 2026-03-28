# SHRAV Scoring & Selection Guide

## Step 1: Tag Each Candidate Post

For each candidate post from the harvest, infer 1–5 short topic tags in Japanese (e.g., `旅行`, `ゲーム`, `猫`, `Python`, `映画`). These tags are stored in `shrav-feedback.md` under `topic_tags` and used for profile derivation.

## Step 2: Score Each Post

Assign a score from -2 to +3 based on the user profile in `shrav-profiling.md`:

| Condition | Score delta |
|-----------|------------|
| Tags match an **Interest** topic | +2 |
| Tags match a **Dislike** topic | -2 |
| Tags are **Neutral** (no profile signal) | +1 (serendipity bonus) |
| Message has rich content (emoji, images, links, multi-sentence) | +0.5 |
| Message is very short / one-liner with no substance | -0.5 |
| Posted by someone in a channel the user hasn't joined | +0.5 (discovery bonus) |

If no profile exists yet (first run), treat all posts as Neutral (+1) and pick the most engaging-looking ones.

## Step 3: Select Posts

1. Sort by score descending.
2. Take up to `max_topics` posts.
3. **But ensure diversity**: no more than 2 posts from the same channel, and no more than 2 posts on the same dominant tag. Demote extras to make room for variety.
4. Reserve **at least 1 slot** (out of max_topics) for a Neutral/serendipity pick, even if high-scoring Interest posts could fill all slots. This keeps the feed fresh.

## Step 4: Build Recommendations List

For each selected post, prepare:
- `permalink` — for the button action_id and the block link
- `channel_name` — for display
- `author_name` — from memberInfo (display name)
- `text_preview` — first 100 characters of the message text
- `score` — for debugging (not shown to user)
- `topic_tags` — for feedback logging

---

## Profile Derivation (Workflow 3)

After pruning feedback, re-derive the profile:

1. **Collect all +1 posts** → extract their `topic_tags` → tally frequency → top tags become **Interests**
2. **Collect all -1 posts** → extract their `topic_tags` → tally frequency → top tags become **Dislikes**
3. **Tags with no clear +1/-1 signal** → list under **Neutral / Serendipity candidates**
4. Write qualitative notes: look at actual message text of top +1 posts and describe what they have in common beyond just tags (tone, format, personal vs. informational, etc.)

**Minimum data threshold:** If there are fewer than 5 feedback entries total, write a minimal profile noting "データが少ないため、全ジャンル均等に探索中" and skip the Interests/Dislikes sections.
