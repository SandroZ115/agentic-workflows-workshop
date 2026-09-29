---
name: Daily Digest
on:
  schedule: daily on weekdays
  workflow_dispatch:
permissions:
  contents: read
  issues: read
  pull-requests: read
engine:
  id: copilot
  model: gpt-4.1
tools:
  github:
    toolsets: [issues, pull_requests]
safe-outputs:
  create-issue:
    max: 1
---

# Daily Digest

Create a GitHub issue that summarises all open issues and pull requests in this repository.

- Group them by label.
- Include the total count, and for each item: the title, the author, and how long it has been open.
- Title the issue "Daily Digest - <today's date>".
- If there are no open issues or pull requests, say so in the issue.