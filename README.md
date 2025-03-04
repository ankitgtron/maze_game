
## **Escape the Maze - Mini Game Documentation**

### **Project Overview**

**Escape the Maze** is a simple game where players navigate a small character from the start position (`🟢`) to the exit (`🏁`). The maze layout is stored in a **SQLite database**, and the player’s time is recorded in a **leaderboard**. The challenge is to escape the maze in the shortest possible time.

---

## **Tech Stack**

- **Frontend**: HTML, CSS, JavaScript (Canvas API)
- **Backend**: Node.js with Express.js
- **Database**: SQLite (stores maze layout & leaderboard)

---

## **Features**

- Players can move using **arrow keys**.
- **Obstacles** in the maze prevent direct movement.
- **Timer** tracks completion time.
- **Leaderboard** stores the best scores.
- **Scalable**: More maze levels can be added.

---

## **Project Structure**

```
/escape-maze-game
 ├── /public
 │   ├── index.html
 │   ├── style.css
 │   ├── script.js
 ├── server.js
 ├── database.sqlite
 ├── package.json
```

---

## **Setup Instructions**

### **1. Clone the Repository**

```bash
git clone https://github.com/your-repo/escape-maze-game.git
cd escape-maze-game
```

### **2. Install Dependencies**

```bash
npm install express sqlite3
```

### **3. Start the Backend Server**

```bash
node server.js
```

### **4. Open the Game**

Open `http://localhost:3000` in your browser.

---

## **Backend - `server.js`**

The backend is built using **Node.js** and **Express.js**. It uses **SQLite** to store the maze layout and leaderboard.

```javascript
const express = require("express");
const sqlite3 = require("sqlite3").verbose();

const app = express();
app.use(express.json());

// Connect to SQLite database (file-based)
const db = new sqlite3.Database("./database.sqlite");

// Initialize database tables
db.serialize(() => {
    // Create maze table if it doesn't exist
    db.run(`
        CREATE TABLE IF NOT EXISTS maze (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            layout TEXT
        )
    `);

    // Insert default maze layout
    db.run(`
        INSERT INTO maze (layout)
        SELECT '[["S",0,1,0,"E"],[0,1,0,0,0],[0,0,0,1,0],[1,0,1,0,0],[0,0,0,0,0]]'
        WHERE NOT EXISTS (SELECT 1 FROM maze WHERE id = 1)
    `);

    // Create leaderboard table if it doesn't exist
    db.run(`
        CREATE TABLE IF NOT EXISTS leaderboard (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            player_name TEXT,
            time_taken FLOAT
        )
    `);
});

// API to fetch maze layout
app.get("/maze", (req, res) => {
    db.get("SELECT layout FROM maze WHERE id = 1", (err, row) => {
        if (err) {
            return res.status(500).json({ error: err.message });
        }
        res.json({ layout: JSON.parse(row.layout) });
    });
});

// API to save player score to leaderboard
app.post("/leaderboard", (req, res) => {
    const { player_name, time_taken } = req.body;
    db.run(
        "INSERT INTO leaderboard (player_name, time_taken) VALUES (?, ?)",
        [player_name, time_taken],
        (err) => {
            if (err) {
                return res.status(500).json({ error: err.message });
            }
            res.json({ message: "Score saved!" });
        }
    );
});

// API to fetch leaderboard
app.get("/leaderboard", (req, res) => {
    db.all("SELECT player_name, time_taken FROM leaderboard ORDER BY time_taken ASC", (err, rows) => {
        if (err) {
            return res.status(500).json({ error: err.message });
        }
        res.json({ leaderboard: rows });
    });
});

// Serve static files from the "public" directory
app.use(express.static("public"));

// Start the server
app.listen(3000, () => {
    console.log("Server running on http://localhost:3000");
});
```

---

## **Frontend - `index.html`**

The frontend is built using **HTML**, **CSS**, and **JavaScript**. It uses the **Canvas API** to render the maze.

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Escape the Maze</title>
    <link rel="stylesheet" href="style.css">
</head>
<body>
    <h2>Escape the Maze</h2>
    <canvas id="mazeCanvas"></canvas>
    <p id="timer">Time: 0s</p>
    <input type="text" id="playerName" placeholder="Enter Name">
    <button onclick="submitScore()">Submit Score</button>
    <h3>Leaderboard</h3>
    <ul id="leaderboard"></ul>
    <script src="script.js"></script>
</body>
</html>
```

---

## **Frontend - `style.css`**

Simple styling for the game.

```css
body {
    text-align: center;
    font-family: Arial, sans-serif;
}

canvas {
    border: 2px solid black;
    margin: 20px auto;
}

input, button {
    margin: 10px;
    padding: 5px;
}

#leaderboard {
    list-style-type: none;
    padding: 0;
}
```

---

## **Frontend - `script.js`**

Handles game logic, including player movement, timer, and API calls.

```javascript
const canvas = document.getElementById("mazeCanvas");
const ctx = canvas.getContext("2d");
canvas.width = 250;
canvas.height = 250;

let maze = [];
let player = { x: 0, y: 0 };
let timer = 0;
let interval;

// Load maze layout from backend
async function loadMaze() {
    const res = await fetch("http://localhost:3000/maze");
    const data = await res.json();
    maze = data.layout;
    drawMaze();
    startTimer();
    loadLeaderboard();
}

// Draw maze on canvas
function drawMaze() {
    ctx.clearRect(0, 0, canvas.width, canvas.height);
    const cellSize = 50;
    maze.forEach((row, y) => {
        row.forEach((cell, x) => {
            if (cell === 1) ctx.fillStyle = "black";
            else if (cell === "S") {
                player = { x, y };
                ctx.fillStyle = "green";
            } else if (cell === "E") ctx.fillStyle = "red";
            else ctx.fillStyle = "white";
            ctx.fillRect(x * cellSize, y * cellSize, cellSize, cellSize);
        });
    });
}

// Handle player movement
document.addEventListener("keydown", (e) => {
    let { x, y } = player;
    if (e.key === "ArrowUp" && y > 0 && maze[y - 1][x] !== 1) player.y--;
    if (e.key === "ArrowDown" && y < 4 && maze[y + 1][x] !== 1) player.y++;
    if (e.key === "ArrowLeft" && x > 0 && maze[y][x - 1] !== 1) player.x--;
    if (e.key === "ArrowRight" && x < 4 && maze[y][x + 1] !== 1) player.x++;
    drawMaze();
    ctx.fillStyle = "green";
    ctx.fillRect(player.x * 50, player.y * 50, 50, 50);
    if (maze[player.y][player.x] === "E") {
        clearInterval(interval);
        alert(`Escaped in ${timer} seconds!`);
    }
});

// Start timer
function startTimer() {
    timer = 0;
    interval = setInterval(() => {
        timer += 1;
        document.getElementById("timer").innerText = `Time: ${timer}s`;
    }, 1000);
}

// Submit score to leaderboard
async function submitScore() {
    const playerName = document.getElementById("playerName").value;
    if (!playerName) return alert("Enter your name first!");
    await fetch("http://localhost:3000/leaderboard", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ player_name: playerName, time_taken: timer })
    });
    alert("Score submitted!");
    loadLeaderboard();
}

// Load leaderboard
async function loadLeaderboard() {
    const res = await fetch("http://localhost:3000/leaderboard");
    const data = await res.json();
    const leaderboard = document.getElementById("leaderboard");
    leaderboard.innerHTML = data.leaderboard
        .map((entry) => `<li>${entry.player_name}: ${entry.time_taken}s</li>`)
        .join("");
}

// Initialize game
loadMaze();
```

---

## **How to Play**

1. Use the **arrow keys** to move the green square (`🟢`) to the red square (`🏁`).
2. Avoid the black squares (`⬛`), which are obstacles.
3. The timer starts when the game loads and stops when you reach the exit.
4. Enter your name and submit your score to the leaderboard.

---

## **Leaderboard**

The leaderboard displays the top scores in ascending order (fastest time first). It updates automatically when a new score is submitted.

---

## **Database Schema**

### **Maze Table**

| Column | Type    | Description          |
|--------|---------|----------------------|
| id     | INTEGER | Primary key          |
| layout | TEXT    | Maze layout (JSON)   |

### **Leaderboard Table**

| Column      | Type    | Description          |
|-------------|---------|----------------------|
| id          | INTEGER | Primary key          |
| player_name | TEXT    | Player's name        |
| time_taken  | FLOAT   | Time taken to escape |

---

## **Future Enhancements**

1. **Multiple Maze Levels**: Add more maze layouts and allow players to choose a level.
2. **Player Authentication**: Allow players to log in and track their scores.
3. **Improved UI**: Add animations, sound effects, and a better design.
4. **Mobile Support**: Make the game responsive for mobile devices.

---
