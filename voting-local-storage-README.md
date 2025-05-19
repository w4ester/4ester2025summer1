# Local Storage with Export/Import

## Overview
This approach uses browser localStorage for voting and provides export/import functionality for sharing results.

## How It Works

### Diagram
```
User Votes
    |
    v
Store in localStorage
    |
    +----> Display Results
    |
    +----> Export to JSON File
            |
            v
        Download to User
            |
            v
        Admin Uploads to GitHub
            |
            v
        Other Users Import JSON
            |
            v
        Merge with Local Votes
```

### Implementation

```javascript
// Enhanced voting system with export/import
class VotingSystem {
  constructor() {
    this.storageKey = 'tripVotes';
    this.loadVotes();
  }

  loadVotes() {
    const stored = localStorage.getItem(this.storageKey);
    this.votes = stored ? JSON.parse(stored) : {
      'Clearwater': 0,
      'Outer Banks': 0,
      'Smokies': 0,
      'lastUpdated': new Date().toISOString()
    };
  }

  saveVotes() {
    this.votes.lastUpdated = new Date().toISOString();
    localStorage.setItem(this.storageKey, JSON.stringify(this.votes));
  }

  vote(tripName) {
    if (this.votes[tripName] !== undefined) {
      this.votes[tripName]++;
      this.saveVotes();
      this.updateDisplay();
    }
  }

  exportVotes() {
    const dataStr = JSON.stringify(this.votes, null, 2);
    const dataBlob = new Blob([dataStr], { type: 'application/json' });
    const url = URL.createObjectURL(dataBlob);
    
    const link = document.createElement('a');
    link.href = url;
    link.download = `votes_${new Date().toISOString().split('T')[0]}.json`;
    document.body.appendChild(link);
    link.click();
    document.body.removeChild(link);
    URL.revokeObjectURL(url);
  }

  async importVotes(file) {
    try {
      const text = await file.text();
      const importedVotes = JSON.parse(text);
      
      // Merge strategy: add imported votes to existing
      for (const trip in importedVotes) {
        if (trip !== 'lastUpdated' && this.votes[trip] !== undefined) {
          this.votes[trip] += importedVotes[trip];
        }
      }
      
      this.saveVotes();
      this.updateDisplay();
      alert('Votes imported successfully!');
    } catch (error) {
      alert('Error importing votes: ' + error.message);
    }
  }

  async fetchPublicVotes() {
    try {
      const response = await fetch('public-votes.json');
      const publicVotes = await response.json();
      return publicVotes;
    } catch (error) {
      console.log('No public votes file found');
      return null;
    }
  }

  updateDisplay() {
    const resultsDiv = document.getElementById('results');
    resultsDiv.innerHTML = Object.entries(this.votes)
      .filter(([key]) => key !== 'lastUpdated')
      .map(([trip, count]) => `<li>${trip}: ${count} votes</li>`)
      .join('');
  }
}

// Initialize voting system
const votingSystem = new VotingSystem();

// Add UI elements to HTML
document.body.insertAdjacentHTML('beforeend', `
  <div class="voting-controls" style="text-align: center; margin: 20px;">
    <button onclick="votingSystem.exportVotes()">Export Votes</button>
    <input type="file" id="importFile" accept=".json" style="display: none;">
    <button onclick="document.getElementById('importFile').click()">Import Votes</button>
  </div>
`);

// Handle file import
document.getElementById('importFile').addEventListener('change', (event) => {
  const file = event.target.files[0];
  if (file) {
    votingSystem.importVotes(file);
  }
});
```

### Admin Workflow

1. **Collect Votes**: Periodically export votes from your browser
2. **Merge Files**: Combine multiple export files
3. **Update Repository**: Commit merged `public-votes.json` to repo
4. **Automated Merge Script**:

```javascript
// merge-votes.js - Run locally or in GitHub Action
const fs = require('fs');
const path = require('path');

function mergeVoteFiles(directory) {
  const files = fs.readdirSync(directory)
    .filter(file => file.endsWith('.json'));
  
  let mergedVotes = {
    'Clearwater': 0,
    'Outer Banks': 0,
    'Smokies': 0
  };
  
  files.forEach(file => {
    const data = JSON.parse(fs.readFileSync(path.join(directory, file)));
    for (const trip in data) {
      if (trip !== 'lastUpdated' && mergedVotes[trip] !== undefined) {
        mergedVotes[trip] += data[trip];
      }
    }
  });
  
  mergedVotes.lastUpdated = new Date().toISOString();
  fs.writeFileSync('public-votes.json', JSON.stringify(mergedVotes, null, 2));
}

mergeVoteFiles('./vote-exports');
```

## Security Considerations

### Architecture
```
[Browser A] --> [localStorage] --> [Export JSON] --> [Admin]
                                                       |
[Browser B] --> [localStorage] --> [Export JSON] ------+
                                                       |
[Browser C] <-- [Import] <-- [public-votes.json] <-----+
```

### Vulnerabilities
1. **Client-Side Manipulation**: Users can modify localStorage
2. **Multiple Votes**: No prevention of multiple votes per user
3. **Trust-Based System**: Relies on honest vote submission
4. **No User Tracking**: Cannot identify unique voters

### Mitigation Strategies
1. **Digital Signatures**: Sign exported votes with crypto API
```javascript
async function signVotes(votes) {
  const encoder = new TextEncoder();
  const data = encoder.encode(JSON.stringify(votes));
  const hash = await crypto.subtle.digest('SHA-256', data);
  return Array.from(new Uint8Array(hash))
    .map(b => b.toString(16).padStart(2, '0'))
    .join('');
}
```

2. **Fingerprinting**: Add browser fingerprint to prevent duplicates
```javascript
function getBrowserFingerprint() {
  const canvas = document.createElement('canvas');
  const ctx = canvas.getContext('2d');
  ctx.textBaseline = 'top';
  ctx.font = '14px Arial';
  ctx.fillText('fingerprint', 2, 2);
  return canvas.toDataURL();
}
```

3. **Vote Validation**: Add timestamps and validate vote patterns
```javascript
function validateVote(voteData) {
  // Check if vote is within reasonable time window
  const voteTime = new Date(voteData.timestamp);
  const now = new Date();
  const hoursDiff = (now - voteTime) / (1000 * 60 * 60);
  
  return hoursDiff < 24 && hoursDiff > 0;
}
```

## Pros
- Simple implementation
- No external dependencies
- Works offline
- User controls their data
- Transparent process

## Cons
- Manual vote aggregation
- Trust-based system
- No real-time updates
- Requires admin intervention

## Advanced Features

### Auto-Import Public Votes
```javascript
// Check for public votes on page load
window.addEventListener('load', async () => {
  const publicVotes = await votingSystem.fetchPublicVotes();
  if (publicVotes) {
    // Show option to merge public votes
    if (confirm('Import latest public vote counts?')) {
      votingSystem.importVotes(new Blob([JSON.stringify(publicVotes)]));
    }
  }
});
```

### Encrypted Exports
```javascript
async function encryptVotes(votes, password) {
  const encoder = new TextEncoder();
  const data = encoder.encode(JSON.stringify(votes));
  
  const key = await crypto.subtle.importKey(
    'raw',
    encoder.encode(password),
    'AES-GCM',
    false,
    ['encrypt']
  );
  
  const iv = crypto.getRandomValues(new Uint8Array(12));
  const encrypted = await crypto.subtle.encrypt(
    { name: 'AES-GCM', iv },
    key,
    data
  );
  
  return { encrypted, iv };
}
```