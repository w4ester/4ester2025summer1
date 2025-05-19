# Static JSON with GitHub Actions

## Overview
This approach stores votes in a JSON file within the repository and uses GitHub Actions to update it securely.

## How It Works

### Diagram
```
User Votes (Frontend)
    |
    v
Send Vote to GitHub API
    |
    v
Triggers GitHub Action (workflow_dispatch)
    |
    v
GitHub Action Updates votes.json
    |
    v
Commits Updated File
    |
    v
GitHub Pages Serves Updated JSON
    |
    v
Frontend Fetches & Displays Results
```

### Implementation Steps

1. Create `votes.json` in your repository:
```json
{
  "votes": {
    "Clearwater": 0,
    "Outer Banks": 0,
    "Smokies": 0
  },
  "lastUpdated": "2024-01-01T00:00:00Z"
}
```

2. Create `.github/workflows/update-votes.yml`:
```yaml
name: Update Votes

on:
  workflow_dispatch:
    inputs:
      trip:
        description: 'Trip name to vote for'
        required: true
        type: choice
        options:
          - Clearwater
          - Outer Banks
          - Smokies

jobs:
  update-votes:
    runs-on: ubuntu-latest
    permissions:
      contents: write
      
    steps:
    - uses: actions/checkout@v3
    
    - name: Update votes
      run: |
        # Read current votes
        VOTES=$(cat votes.json)
        
        # Increment the vote for the selected trip
        TRIP="${{ github.event.inputs.trip }}"
        CURRENT=$(echo $VOTES | jq -r ".votes[\"$TRIP\"]")
        NEW_COUNT=$((CURRENT + 1))
        
        # Update JSON
        UPDATED=$(echo $VOTES | jq ".votes[\"$TRIP\"] = $NEW_COUNT | .lastUpdated = \"$(date -Is)\"")
        echo "$UPDATED" > votes.json
        
    - name: Commit changes
      run: |
        git config --local user.email "action@github.com"
        git config --local user.name "GitHub Action"
        git add votes.json
        git commit -m "Update vote for ${{ github.event.inputs.trip }}"
        git push
```

3. Frontend code to trigger the action:
```javascript
const GITHUB_TOKEN = 'your_token_here'; // Needs workflow scope
const REPO_OWNER = 'w4ester';
const REPO_NAME = '4ester2025summer1';

async function submitVote(tripName) {
  const response = await fetch(
    `https://api.github.com/repos/${REPO_OWNER}/${REPO_NAME}/actions/workflows/update-votes.yml/dispatches`,
    {
      method: 'POST',
      headers: {
        'Authorization': `token ${GITHUB_TOKEN}`,
        'Accept': 'application/vnd.github.v3+json',
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        ref: 'main',
        inputs: {
          trip: tripName
        }
      })
    }
  );
  
  if (response.ok) {
    console.log('Vote submitted');
    // Wait a moment for the action to complete
    setTimeout(() => fetchVotes(), 5000);
  }
}

async function fetchVotes() {
  const response = await fetch('votes.json');
  const data = await response.json();
  updateVoteDisplay(data.votes);
}
```

## Security Considerations

### Vulnerabilities
1. **Token Exposure**: GitHub token visible in frontend
2. **Workflow Permissions**: Token needs workflow dispatch permissions
3. **Delayed Updates**: Actions take time to run
4. **Git History**: All votes are tracked in commit history

### Mitigation Strategies
1. **Server Proxy**: Hide token behind a server endpoint
2. **Action Validation**: Add validation in the GitHub Action
3. **Branch Protection**: Protect main branch, votes only via Actions
4. **Scheduled Cleanup**: Periodically squash vote commits

### Security Architecture
```
[User] --> [Proxy Server] --> [GitHub API] --> [GitHub Action]
                  |
                  v
            [Hidden Token]
```

## Pros
- Votes stored in version control
- Transparent voting history
- No external services needed
- Free within Actions limits

## Cons
- Complex setup
- Requires GitHub token
- Delayed vote updates
- Potential for rate limiting

## Advanced Options

### Using Issues as Triggers
Instead of workflow_dispatch, create issues to trigger votes:
```yaml
on:
  issues:
    types: [opened]
    
jobs:
  process-vote:
    if: startsWith(github.event.issue.title, 'VOTE:')
    # ... rest of action
```

### Vote Batching
Collect votes and update in batches to reduce commits:
```yaml
on:
  schedule:
    - cron: '*/15 * * * *'  # Every 15 minutes
```