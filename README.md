# 우리 가족 체크리스트 앱 (Family Checklist PWA)

A Progressive Web App for tracking family rules for two children (Midam & Kangho), with Google Drive backup.

## Features implemented

- **Two children tabs**: Midam (daughter, 7 rules) and Kangho (son, 7 rules)
- **Daily checklist**: Each rule can be marked "지켰어요" (Kept) or "못 지켰어요" (Broken)
- **Parent comment**: When a rule is marked as broken, parent can add a note explaining what happened
- **Cumulative punishment system**:
  - Day 1 broken → 1 day punishment
  - Day 2 also broken → adds 2 more (total 3 days)
  - Day 3 also broken → adds 3 more (total 6 days)
  - Formula: `N × (N+1) / 2` where N = consecutive broken days
  - All three banned together: 간식 + 폰 + TV
- **7-day history report**: Visual dot summary showing which rules kept/broken each day
- **Study log**: Three types matching your spec
  - 숙제/시험 (Homework/Test): unit, date, time, page, start/end time
  - 연습장 (Practice): unit, time, sentences, page
  - 마음의 소리 (Mind voice): daily journal (daughter only)
- **Parent PIN**: 4-digit PIN to protect override actions
- **Offline-first**: Works without internet using localStorage
- **Installable**: Add to home screen → looks like a native app
- **Backup**: Export JSON file; Google Drive sync (requires OAuth setup)

## Quick start — run on your Galaxy phone today

### Option A: Host on GitHub Pages (recommended, free, 5 minutes)

1. Create a new GitHub repository (public, free)
2. Upload all files from this folder (`index.html`, `manifest.json`, `sw.js`, `icon-192.png`, `icon-512.png`)
3. Go to repository **Settings → Pages → Source: main branch**
4. GitHub gives you a URL like `https://yourname.github.io/reponame/`
5. On your Galaxy phone, open that URL in Chrome
6. Tap menu (⋮) → **"홈 화면에 추가"** (Add to Home screen)
7. App icon appears on your home screen — tap it and it runs like a native app

### Option B: Test locally first on your PC

```bash
cd family-app
python -m http.server 8000
# Open http://localhost:8000 in your browser
```

Then to access from your Galaxy phone on the same Wi-Fi:
```bash
# Find your PC's IP (on Windows PowerShell)
ipconfig
# Look for IPv4 Address, e.g. 192.168.0.10
# On phone, visit http://192.168.0.10:8000
```

**Note**: PWA "Add to Home Screen" requires HTTPS. GitHub Pages provides HTTPS automatically. For local testing, you can use `ngrok` or skip the installability step.

## Google Drive integration

The backup button currently downloads a JSON file. To connect directly to Google Drive, you need to:

### Step 1: Create Google Cloud project

1. Go to https://console.cloud.google.com/
2. Create a new project (e.g., "FamilyChecklist")
3. Enable **Google Drive API** under "APIs & Services"
4. Go to **Credentials → Create Credentials → OAuth 2.0 Client ID**
5. Application type: **Web application**
6. Authorized JavaScript origins: Add your hosting URL (e.g., `https://yourname.github.io`)
7. Copy the **Client ID**

### Step 2: Add Google Identity Services code

In `index.html`, find the `connectGoogleDrive()` function and replace it with this (after adding the Client ID):

```javascript
const CLIENT_ID = 'YOUR_CLIENT_ID_HERE.apps.googleusercontent.com';
const SCOPES = 'https://www.googleapis.com/auth/drive.appdata';

// Add to <head> of index.html:
// <script src="https://accounts.google.com/gsi/client" async defer></script>
// <script src="https://apis.google.com/js/api.js" async defer></script>

let tokenClient;
let accessToken = null;

function initGoogleDrive() {
  tokenClient = google.accounts.oauth2.initTokenClient({
    client_id: CLIENT_ID,
    scope: SCOPES,
    callback: (resp) => {
      accessToken = resp.access_token;
      window.driveConnected = true;
      updateSyncStatus('구글 드라이브 연결됨');
      backupToDrive();
    }
  });
}

function connectGoogleDrive() {
  if (!tokenClient) initGoogleDrive();
  tokenClient.requestAccessToken();
}

async function backupToDrive() {
  if (!accessToken) return;
  const fileName = `family_checklist_${todayStr()}.json`;
  const content = JSON.stringify(state.data);
  
  // Search for existing file
  const searchRes = await fetch(
    `https://www.googleapis.com/drive/v3/files?spaces=appDataFolder&q=name='${fileName}'`,
    { headers: { Authorization: `Bearer ${accessToken}` }}
  );
  const search = await searchRes.json();
  
  const metadata = { name: fileName, parents: ['appDataFolder'] };
  const form = new FormData();
  form.append('metadata', new Blob([JSON.stringify(metadata)], {type: 'application/json'}));
  form.append('file', new Blob([content], {type: 'application/json'}));
  
  let url = 'https://www.googleapis.com/upload/drive/v3/files?uploadType=multipart';
  let method = 'POST';
  if (search.files && search.files[0]) {
    url = `https://www.googleapis.com/upload/drive/v3/files/${search.files[0].id}?uploadType=multipart`;
    method = 'PATCH';
  }
  
  await fetch(url, {
    method,
    headers: { Authorization: `Bearer ${accessToken}` },
    body: form
  });
  updateSyncStatus('드라이브 백업됨 · ' + new Date().toLocaleTimeString('ko'));
}
```

The `appDataFolder` scope means the data is stored in a hidden folder only this app can access — safer for kids' data privacy.

## Architecture summary

```
┌─────────────────────────────────────┐
│  Galaxy phone (PWA installed)       │
│  ┌──────────────────────────────┐   │
│  │  HTML/CSS/JS (single file)   │   │
│  │  + Service Worker (offline)  │   │
│  └──────────────┬───────────────┘   │
│                 │                   │
│  ┌──────────────▼───────────────┐   │
│  │  localStorage (daily data)   │   │
│  └──────────────┬───────────────┘   │
└─────────────────┼───────────────────┘
                  │ (5s debounced sync)
                  ▼
┌─────────────────────────────────────┐
│  Google Drive appDataFolder         │
│  family_checklist_2026-04-21.json   │
│  (per-day JSON, auto-merged)        │
└─────────────────────────────────────┘
```

## Data format (stored as JSON)

```json
{
  "midam_2026-04-21": {
    "rules": { "0": "kept", "1": "broken", "2": "kept", ... },
    "comments": { "1": "동생이랑 아침부터 싸웠음. 내일은 먼저 양보하기로 약속" },
    "studies": [
      {
        "type": "homework",
        "unit": "Unit 5",
        "page": "32",
        "start": "14:00",
        "end": "14:45",
        "content": "영어 단어 외우기"
      }
    ]
  },
  "kangho_2026-04-21": { ... }
}
```

## Future enhancements (if needed)

- Weekly/monthly PDF report for review sessions
- Push notifications (reminder at 7pm to fill out today's checklist)
- Photo attachment (picture of completed homework)
- Migration to MAUI native app once PWA proves the logic

## Why this is the right solution for you

Given your VB.NET + IoT background, this PWA mirrors patterns you already know:
- **localStorage = local SQLite buffer on PLC edge device** (offline-first, sync when connected)
- **Daily JSON files to Drive = timestamped log files to server** (same pattern as PLC data archival)
- **Service worker = background daemon** (runs independently of main app)
- **The cumulative punishment formula is just triangular numbers** (1 + 2 + 3 + ... + N = N(N+1)/2)

If the PWA proves the UX works well with your kids, porting to a MAUI C# native app later is straightforward — the data model and punishment logic transfer directly.
