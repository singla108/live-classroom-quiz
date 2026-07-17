# Usage Guide

## Quick Start

### Prerequisites
- Node.js 18+ installed on your machine
- A device to project the host screen (laptop/projector)
- Students with smartphones on the same network (for local use) or internet access (for hosted use)

### Installation

```bash
cd Quiz
npm install
```

### Running

```bash
npm start
```

Output:
```
🎯 Quiz App Server Running!
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Host Dashboard: http://localhost:3000
  Student Join:   http://192.168.x.x:3000/student.html
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## Running a Quiz Session

### Step 1: Select a Quiz

1. Open `http://localhost:3000` on the projector/host screen
2. You'll see a **quiz selection screen** with buttons for each batch
3. Click **Batch A** or **Batch B** to load that question set

### Step 2: Student Login via QR Code

1. After selecting a quiz, the **lobby screen** appears with:
   - A large QR code
   - The join URL displayed below it
2. Students scan the QR code with their phone camera
3. They enter their chosen display name and tap **Join Quiz**
4. Names appear as chips on the host screen in real-time

### Step 3: Start the Quiz

1. Once all students have joined (player count shown on screen), click **Start Quiz**
2. The first question appears immediately on all screens

### Step 4: During Each Question

**Host screen shows:**
- Question text and 4 color-coded options
- Countdown timer
- Response counter (X / Y answered)
- Word cloud of student names as they respond (animates in)

**Student screen shows:**
- Question text
- 4 tappable option buttons (color-coded)
- Countdown timer with urgency indicator at 5 seconds

### Step 5: Show Answer

1. After enough time or all students respond, click **Show Answer**
2. **Host screen:** Highlights the correct answer in green, shows live rankings
3. **Student screen:** Displays the correct answer with a green highlight

### Step 6: Next Question

Click **Next Question** to advance. Repeat steps 4–6 for all questions.

### Step 7: Final Rankings

After the last question, the **final rankings screen** appears automatically:
- **Podium** with top 3 (gold, silver, bronze)
- **Full leaderboard** with rank, name, correct answers, and total score
- Students see their personal rank and score on their phones

### Step 8: Run Another Quiz

Click **New Quiz** to return to the quiz selection screen. Select the other batch and repeat for Batch B.

---

## Customizing Questions

### Question File Format

Edit `questions-batch-a.json` or `questions-batch-b.json`:

```json
{
  "quizTitle": "Your Quiz Title",
  "timePerQuestion": 20,
  "questions": [
    {
      "id": 1,
      "question": "Your question text here?",
      "options": ["Option A", "Option B", "Option C", "Option D"],
      "correctAnswer": 1,
      "timeLimit": 20
    }
  ]
}
```

**Field reference:**

| Field | Type | Description |
|-------|------|-------------|
| `quizTitle` | string | Displayed on the host dashboard header |
| `timePerQuestion` | number | Default seconds per question (fallback) |
| `questions[].id` | number | Unique question identifier |
| `questions[].question` | string | The question text |
| `questions[].options` | string[] | Exactly 4 answer options |
| `questions[].correctAnswer` | number | 0-indexed correct option (0=A, 1=B, 2=C, 3=D) |
| `questions[].timeLimit` | number | Seconds for this specific question (overrides default) |

### Adding a New Quiz Batch

1. Create a new file, e.g. `questions-batch-c.json`
2. Add an entry in `server.js`:

```javascript
const quizFiles = {
  'batch-a': { file: 'questions-batch-a.json', label: 'Batch A' },
  'batch-b': { file: 'questions-batch-b.json', label: 'Batch B' },
  'batch-c': { file: 'questions-batch-c.json', label: 'Batch C' },  // ← add this
};
```

3. Restart the server — the new batch appears on the selection screen.

### Hot-Reloading Questions

If you edit a question file while the server is running (and no quiz is active), call:

```bash
curl -X POST http://localhost:3000/api/reload-questions
```

This reloads the current quiz's questions from disk without restarting the server.

---

## Scoring System

| Condition | Points |
|-----------|--------|
| Wrong answer | 0 |
| Correct answer (at time limit) | 1000 |
| Correct answer (instant) | 1500 |
| Speed bonus formula | `500 × (1 - timeTaken/timeLimit)` |

**Example:** 20-second question, answered correctly in 5 seconds:
- Base: 1000
- Speed bonus: 500 × (1 - 5/20) = 375
- **Total: 1375 points**

**Tiebreaker:** If two students have the same total score, the one with lower cumulative response time ranks higher.

---

## Deployment Options

### Local (Same WiFi Network)

Just run `npm start`. Students must be on the same WiFi network to reach the local IP shown in the QR code.

### Cloud Hosting (Recommended for reliability)

The app works on any Node.js-compatible platform with WebSocket support:

| Platform | Free Tier | Setup |
|----------|-----------|-------|
| **Render** | 750 hrs/month | Connect GitHub → New Web Service → `npm start` |
| **Railway** | $5 credit/month | Connect GitHub → Deploy |
| **Fly.io** | 3 free VMs | `fly launch` → `fly deploy` |
| **Glitch** | Always-on while active | Import from GitHub |

**Environment variables:**
- `PORT` — The port to listen on (platforms set this automatically)

The QR code automatically uses the correct public URL when deployed (reads from request headers).

---

## Troubleshooting

### Students can't connect
- Ensure all devices are on the same WiFi network (local mode)
- Check firewall isn't blocking port 3000
- Try the URL shown below the QR code manually

### QR code shows wrong URL
- On hosted platforms, ensure the app receives `X-Forwarded-Host` and `X-Forwarded-Proto` headers
- Locally, verify your machine's IP hasn't changed

### Timer seems off
- The timer runs client-side for display but scoring uses the server timestamp
- Small network latency doesn't significantly affect scoring (base 1000 points means even 1-second lag costs max ~25 points)

### Students disconnect mid-quiz
- They auto-reconnect within 2 seconds
- Their scores and previous answers are preserved
- If they rejoin on the same question, they can still answer

### Want to restart the same quiz
- Click **New Quiz** on the final rankings screen
- Select the same batch again — all scores and players are cleared

---

## Session Timeline (Recommended)

| Time | Activity |
|------|----------|
| 0:00 | Open host dashboard, select Batch A |
| 0:01 | Display QR code, students join (allow 2–3 min) |
| 0:03 | Start quiz |
| 0:03–0:10 | Run through 12 questions (~7 min) |
| 0:10 | Review final rankings |
| 0:11 | Select Batch B, new QR code (or reuse same session) |
| 0:11–0:13 | Students join Batch B |
| 0:13–0:20 | Run Batch B |
| 0:20 | Final rankings, wrap up |

**Total: ~20 minutes for both batches**
