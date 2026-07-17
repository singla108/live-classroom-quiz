# Technical Flow Document

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        Node.js Server                            │
│                        (server.js)                               │
│                                                                  │
│  ┌──────────────┐   ┌──────────────────┐   ┌────────────────┐  │
│  │  Express HTTP │   │  WebSocket Server │   │  Game State    │  │
│  │  (REST APIs)  │   │  (Real-time)      │   │  (In-memory)   │  │
│  └──────────────┘   └──────────────────┘   └────────────────┘  │
│         │                    │                       │           │
│         │                    │                       │           │
│  ┌──────┴──────────────────┴───────────────────────┴────────┐  │
│  │              Question Files (JSON)                         │  │
│  │   questions-batch-a.json  |  questions-batch-b.json        │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
         │                           │
         │ HTTP                      │ WebSocket
         ▼                           ▼
┌─────────────────┐         ┌─────────────────┐
│  Host Dashboard  │         │  Student Mobile  │
│  (index.html)    │         │  (student.html)  │
│  - Quiz select   │         │  - Login         │
│  - QR display    │         │  - Answer        │
│  - Word cloud    │         │  - Feedback      │
│  - Rankings      │         │  - Score         │
└─────────────────┘         └─────────────────┘
```

## Technology Stack

| Component | Technology | Purpose |
|-----------|-----------|---------|
| Server | Node.js + Express 4.18 | HTTP server, static file serving, REST APIs |
| Real-time | ws 8.16 (WebSocket) | Bidirectional communication between host/students |
| QR Code | qrcode 1.5 | Generates QR code data URL for student login |
| Frontend | Vanilla HTML/CSS/JS | Zero-dependency client, works on any browser |

## File Structure

```
Quiz/
├── server.js                  # Main server (Express + WebSocket)
├── questions-batch-a.json     # Batch A question bank
├── questions-batch-b.json     # Batch B question bank
├── questions.json             # Default/legacy question bank
├── package.json               # Dependencies and scripts
└── public/
    ├── index.html             # Host dashboard (projector screen)
    ├── student.html           # Student mobile interface
    └── styles.css             # All styling
```

## Data Flow

### 1. Quiz Selection Phase

```
Host Browser                    Server
    │                              │
    │── GET /api/quizzes ─────────►│  Returns list of available quiz files
    │◄─ [{id, label}, ...] ───────│
    │                              │
    │── POST /api/select-quiz ────►│  Loads questions, resets game state
    │◄─ {title, totalQuestions} ───│
    │                              │
    │── GET /api/qrcode ──────────►│  Generates QR pointing to student.html
    │◄─ {qr: dataURL, url} ───────│
    │                              │
    │── WS: {type: 'host-join'} ──►│  Registers as host role
    │◄─ {type: 'game-state'} ─────│
```

### 2. Student Join Phase

```
Student Phone                   Server                    Host Browser
    │                              │                           │
    │── Scan QR → student.html     │                           │
    │── WS: {type:'player-join',   │                           │
    │    name:'Alice'}  ──────────►│                           │
    │                              │── WS: {type:'player-joined',
    │                              │    name:'Alice',           │
    │◄─ {type:'join-confirmed',    │    players:[...]} ───────►│
    │    playerId:'player_xxx'}    │                           │
```

### 3. Quiz Active Phase (Per Question)

```
Host                          Server                     Students
 │                              │                           │
 │── WS: 'start-quiz' ────────►│                           │
 │                              │── WS: 'quiz-started' ───►│
 │                              │                           │
 │◄─ WS: 'question-host' ─────│── WS: 'question' ───────►│
 │   (includes correctAnswer)  │   (options only)          │
 │                              │                           │
 │                              │◄─ WS: 'submit-answer' ──│  Student taps option
 │                              │── WS: 'answer-confirmed'►│  Score feedback
 │◄─ WS: 'player-responded'───│                           │
 │   (word cloud update)        │                           │
 │                              │                           │
 │── WS: 'show-results' ──────►│                           │
 │                              │── WS: 'question-results'►│  Shows correct answer
 │◄─ WS: 'question-results' ──│                           │
 │   (rankings table)           │                           │
 │                              │                           │
 │── WS: 'next-question' ─────►│   (cycle repeats)        │
```

### 4. Quiz End Phase

```
Server detects last question answered:
    │── WS: 'quiz-finished' ──────► All clients
    │   {rankings: [{name, score, correctAnswers, totalTime}, ...]}
```

## Scoring Algorithm

```javascript
function calculateScore(correct, timeTaken, timeLimit) {
  if (!correct) return 0;
  // Base: 1000 points for correct answer
  // Speed bonus: up to 500 points (linear decay with time)
  const timeBonus = Math.round(500 * (1 - timeTaken / timeLimit));
  return 1000 + Math.max(0, timeBonus);
}
```

**Scoring breakdown:**
- Wrong answer: 0 points
- Correct, answered at deadline: 1000 points
- Correct, answered instantly: 1500 points
- Tiebreaker: total cumulative response time (lower = better)

## Ranking Logic

Players are sorted by:
1. **Score (descending)** — more points = higher rank
2. **Total time (ascending)** — faster overall = tiebreaker

## WebSocket Message Types

### Client → Server

| Type | Sender | Payload | Description |
|------|--------|---------|-------------|
| `host-join` | Host | — | Register as host |
| `player-join` | Student | `{name, playerId?}` | Join or rejoin game |
| `start-quiz` | Host | — | Begin the quiz |
| `next-question` | Host | — | Advance to next question |
| `submit-answer` | Student | `{answer: 0-3}` | Submit selected option index |
| `show-results` | Host | — | Reveal correct answer to all |
| `end-quiz` | Host | — | Manually end quiz early |
| `reset-quiz` | Host | — | Reset to lobby for new quiz |

### Server → Client

| Type | Target | Payload | Description |
|------|--------|---------|-------------|
| `game-state` | Host | `{status, players, currentQuestionIndex}` | Initial state sync |
| `join-confirmed` | Student | `{playerId, gameStatus}` | Join acknowledged |
| `player-joined` | Host | `{name, players[]}` | New player notification |
| `player-left` | Host | `{playerId, players[]}` | Disconnection |
| `quiz-started` | All | — | Quiz begins |
| `question` | Students | `{questionIndex, question, options, timeLimit}` | New question |
| `question-host` | Host | `{...question, correctAnswer, totalPlayers}` | Question + answer key |
| `answer-confirmed` | Student | `{correct, score, totalScore}` | Answer feedback |
| `player-responded` | Host | `{name, responsesCount, wordCloudNames[]}` | Word cloud data |
| `question-results` | All | `{question, options, correctAnswer, rankings[]}` | Answer reveal |
| `quiz-finished` | All | `{rankings[]}` | Final leaderboard |
| `quiz-reset` | All | `{players[]}` | Back to lobby |

## REST API Endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/quizzes` | List available quiz files |
| POST | `/api/select-quiz` | Load a specific quiz (`{quizId}`) |
| GET | `/api/qrcode` | Generate QR code for student join URL |
| GET | `/api/quiz` | Get current quiz title and question count |
| POST | `/api/reload-questions` | Hot-reload current quiz file from disk |

## State Management

All state is held **in-memory** on the server:

```javascript
gameState = {
  status: 'waiting' | 'active' | 'finished',
  currentQuestionIndex: number,
  questionStartTime: timestamp,
  players: Map<playerId, {name, score, answers[], ws}>,
  responses: [{playerId, name, time}]  // per-question, reset each question
}
```

**Implications:**
- Server restart = full state loss (acceptable for a session-based quiz)
- No database required
- Supports reconnection (player rejoins with same `playerId`)

## Connection Resilience

- **Heartbeat:** Server pings all clients every 30 seconds; unresponsive clients are terminated
- **Student reconnect:** On WebSocket close, students auto-reconnect after 2 seconds using their stored `playerId`
- **Disconnected players:** Remain in rankings — their `ws` reference is nulled but scores persist

## Security Considerations

- Host-only commands (`start-quiz`, `next-question`, etc.) are gated by `ws.role === 'host'`
- Players cannot submit answers twice per question
- Player IDs are server-generated (timestamp + random string)
- No authentication layer (appropriate for classroom/session use)

## QR Code Generation

The QR code URL is dynamically generated from the incoming HTTP request headers:

```javascript
const protocol = req.headers['x-forwarded-proto'] || req.protocol || 'http';
const host = req.headers['x-forwarded-host'] || req.headers.host || `${LOCAL_IP}:${PORT}`;
const url = `${protocol}://${host}/student.html`;
```

This ensures it works correctly whether running locally or on a cloud platform behind a reverse proxy.
