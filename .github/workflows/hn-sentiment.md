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
      for k in $(jq -r '(.kids // [])[:50][]' /tmp/story.json); do
        curl -s "https://hacker-news.firebaseio.com/v0/item/$k.json"
      done | jq -s '[.[] | select(.text != null and .deleted != true) | {by, text: .text[:500]}]' > /tmp/comments.json
      jq -n --slurpfile s /tmp/story.json --slurpfile c /tmp/comments.json '{id: $s[0].id, title: $s[0].title, comments: $c[0]}' > $OUT
safe-outputs:
  add-comment:
    max: 1
---

# HN Sentiment Analysis

Read /tmp/gh-aw/agent/hn-comments.json. It contains the Hacker News story title and up to 50 top-level comments (author, text).

If the file has an "error" field, or the title is null or there are no comments, reply with a short, helpful error message explaining the command usage: `/hn-sentiment https://news.ycombinator.com/item?id=12345`.

Otherwise:
1. Classify each comment as Positive, Negative, or Neutral.
2. Reply with a Markdown comment that includes: the story title, a table with count and percentage per sentiment, the overall sentiment, the top 3 most positive comments and the top 3 most negative comments (short excerpts, plain text).

Start the reply with "## 🔍 Hacker News Sentiment Analysis".