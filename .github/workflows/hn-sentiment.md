---
name: HN Sentiment Analysis
on:
  slash_command:
    name: hn-sentiment
permissions:
  contents: read
  issues: read
engine:
  id: copilot
  model: gpt-4.1
network:
  allowed:
    - defaults
    - hacker-news.firebaseio.com
tools:
  bash: true
steps:
  - name: Fetch HN comments
    env:
      COMMENT: ${{ github.event.comment.body }}
    run: |
      mkdir -p /tmp/gh-aw/agent
      OUT=/tmp/gh-aw/agent/hn-comments.json
      ID=$(echo "$COMMENT" | grep -oE 'id=[0-9]+' | head -1 | cut -d= -f2)
      if [ -z "$ID" ]; then
        echo '{"error":"no valid Hacker News item URL provided"}' > $OUT
        exit 0
      fi
      curl -s "https://hacker-news.firebaseio.com/v0/item/$ID.json" > /tmp/story.json
      for k in $(jq -r '(.kids // [])[:20][]' /tmp/story.json); do
        curl -s "https://hacker-news.firebaseio.com/v0/item/$k.json"
      done | jq -s '[.[] | select(.text != null and .deleted != true) | {text: .text[:250]}]' > /tmp/comments.json
      jq -n --slurpfile s /tmp/story.json --slurpfile c /tmp/comments.json '{title: $s[0].title, comments: $c[0]}' > $OUT
safe-outputs:
  add-comment:
    max: 1
---

# HN Sentiment Analysis

Read /tmp/gh-aw/agent/hn-comments.json once (a single command). It has the Hacker News story title and up to 20 top-level comments.

If it has an "error" field or no comments, reply with a short usage message: `/hn-sentiment https://news.ycombinator.com/item?id=12345`.

Otherwise, in ONE pass and without further tool calls, classify each comment as Positive, Negative, or Neutral and reply with one Markdown comment containing: the story title, a table with count and percentage per sentiment, the overall sentiment, the top 3 most positive and top 3 most negative comments (short excerpts).

Start the reply with "## 🔍 Hacker News Sentiment Analysis".