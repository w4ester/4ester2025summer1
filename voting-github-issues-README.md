# GitHub Issues as Voting Database

## Overview
This approach uses GitHub Issues comments as a makeshift database for storing votes. Each vote is stored as a comment on a designated issue.

## How It Works

### Diagram
```
User Votes
    |
    v
Frontend JavaScript
    |
    v
GitHub API (using token)
    |
    v
Creates Comment on Issue
    |
    v
Comment Format: "VOTE:Clearwater:timestamp"
    |
    v
On Page Load: Fetch All Comments
    |
    v
Parse & Count Votes
    |
    v
Display Results
```

### Implementation Steps

1. Create a dedicated issue in your repo (e.g., "Vote Tracking - DO NOT CLOSE")
2. Generate a GitHub Personal Access Token with `public_repo` scope
3. Add voting code to your HTML:

```javascript
const GITHUB_TOKEN = 'your_token_here'; // In production, use env variable
const ISSUE_NUMBER = 1; // Your dedicated issue number
const REPO_OWNER = 'w4ester';
const REPO_NAME = '4ester2025summer1';

async function submitVote(tripName) {
  const timestamp = new Date().toISOString();
  const voteData = `VOTE:${tripName}:${timestamp}`;
  
  const response = await fetch(
    `https://api.github.com/repos/${REPO_OWNER}/${REPO_NAME}/issues/${ISSUE_NUMBER}/comments`,
    {
      method: 'POST',
      headers: {
        'Authorization': `token ${GITHUB_TOKEN}`,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({ body: voteData })
    }
  );
  
  if (response.ok) {
    console.log('Vote submitted successfully');
    await fetchAndDisplayVotes();
  }
}

async function fetchAndDisplayVotes() {
  const response = await fetch(
    `https://api.github.com/repos/${REPO_OWNER}/${REPO_NAME}/issues/${ISSUE_NUMBER}/comments`,
    {
      headers: {
        'Authorization': `token ${GITHUB_TOKEN}`,
      }
    }
  );
  
  const comments = await response.json();
  const votes = { 'Clearwater': 0, 'Outer Banks': 0, 'Smokies': 0 };
  
  comments.forEach(comment => {
    if (comment.body.startsWith('VOTE:')) {
      const [_, tripName] = comment.body.split(':');
      if (votes[tripName] !== undefined) {
        votes[tripName]++;
      }
    }
  });
  
  // Display votes
  updateVoteDisplay(votes);
}
```

## Security Considerations

### Vulnerabilities
1. **Exposed API Token**: The GitHub token is visible in client-side code
2. **Token Abuse**: Anyone can use the token for other GitHub API calls
3. **Vote Manipulation**: Users can inspect code and submit multiple votes
4. **Rate Limiting**: GitHub API has rate limits (60 requests/hour unauthenticated, 5000/hour authenticated)

### Mitigation Strategies
1. **Use GitHub OAuth**: Instead of a static token, implement GitHub OAuth flow
2. **Proxy Server**: Create a simple proxy server (e.g., Netlify Functions) to hide the token
3. **Vote Validation**: Add user fingerprinting or require GitHub authentication
4. **Caching**: Cache vote counts to reduce API calls

### Best Practices
- Never commit tokens to your repository
- Use environment variables for sensitive data
- Implement rate limiting on your frontend
- Add CORS headers if using a proxy

## Pros
- No additional services needed
- Votes are transparent (visible in issue comments)
- Version controlled
- Free within API limits

## Cons
- Security concerns with token exposure
- Rate limiting issues with many users
- Not truly anonymous
- Requires GitHub API knowledge

## Alternative: GitHub Actions
You could trigger a GitHub Action on issue comment creation to process votes server-side, improving security.