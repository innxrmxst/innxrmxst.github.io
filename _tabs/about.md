---
# the default layout is 'page'
icon: fas fa-info-circle
order: 4
---

## Maksym Hensitskyi 

Cybersecurity specialist based in Poland 🇵🇱, with over 4 years of experience in both offensive and defensive security.

### Skills & Interests ⚔️

- Penetration Testing
- RedTeam Operations & Adversary Emulation
- Threat Hunting & Incident Response
- System Administration
- Security Automation & Tool Development

### GitHub projects 🛠️ 

- [mssql-relay-lab](https://github.com/innxrmxst/mssql-relay-lab) — A tool for MSSQL relaying to SMB.
- [lurked](https://github.com/innxrmxst/lurked) — A proof-of-concept stealthy agent leveraging QEMU virtualization and a Linux rootkit for process hiding.
- [PAMinant](https://github.com/innxrmxst/PAMinant) — A PAM backdoor toolkit designed for access persistence on Linux systems.

### Certifications 🎓

- [OffSec Certified Professional (OSCP)](https://credentials.offsec.com/1a1b245e-01af-47ca-a35a-e5a55df2e666)
- [Red Team Operator (CRTO)](https://eu.badgr.com/public/assertions/YCOpfKbZSgK0ANr4_AUJYA)
- [Certified Red Team Expert (CRTE)](https://www.credential.net/b7966972-4177-40d2-8482-edd2ed6f49c3)
- [Certified CyberDefender (CCD)](https://www.credly.com/badges/71d0e81f-774f-481c-a29a-df4e65e83017/public_url)
- [Certified Enterprise Security Professional (CESP - ADCS)](https://www.credential.net/0d7479be-ae25-4de7-9998-265f40694685)
- [Certified Azure Red Team Professional (CARTE)](https://www.credential.net/69a107a8-d304-4306-9c5b-8403ac6e3c16)
- [eLearnSecurity Certified Professional Penetration Tester (eCPPT)](https://certs.ine.com/a1925fd0-47dd-4ae9-80ff-384845ba218f)
- [Practical Network Penetration Tester (PNPT)](https://certified.tcm-sec.com/5873d045-021a-4d11-a660-8939cffb0a56)
- [eLearnSecurity Junior Penetration Tester (eJPT)](https://certs.ine.com/b49fa364-654d-462e-8eff-1d325d637505)
- [Certified Red Team Professional (CRTP)](https://www.credential.net/ae71e583-58e0-4c30-b06e-825f3ca4cadf)

### Courses & labs 🧪

- [Linux Privilege Escalation for OSCP & Beyond!](https://www.udemy.com/certificate/UC-757800d6-221d-4a1b-90bc-367117b555ee/)
- [Windows Privilege Escalation for OSCP & Beyond!](https://www.udemy.com/certificate/UC-f730fefc-e5ef-4640-9139-5b8266fe1383/)
- [Introduction to Cyber Security Learning Path](https://tryhackme-certificates.s3-eu-west-1.amazonaws.com/THM-G76XLDCIK5.pdf)
- [Attacking Active Directory with Linux Lab](https://www.credential.net/32a4a63d-b270-4aa5-9a9b-8e1788a7b34c)
- [Wreath Lab](https://tryhackme.com/innxrmxst/badges/wreath)
- [ADVersary Lab](https://tryhackme.com/innxrmxst/badges/adversary)

### Platforms 💻

**HackerOne:**

- [Bypass of restrictions on external links](https://hackerone.com/reports/956449)

**HackTheBox:**

- [hensitskyi](https://app.hackthebox.com/users/222289)
- [innxrmxst](https://app.hackthebox.com/profile/914542)

**TryHackMe:**

- [innxrmxst](https://tryhackme.com/p/innxrmxst)

---

## Play a Mini Game 🎮

<style>
  #snake-game {
    background: #111;
    margin: 20px auto;
    display: block;
    border: 3px solid #444;
    border-radius: 5px;
    box-shadow: 0 0 20px rgba(0, 255, 0, 0.2);
  }
  .game-container {
    text-align: center;
    font-family: Arial, sans-serif;
  }
  #score {
    color: lime;
    font-size: 20px;
    margin-bottom: 10px;
  }
  #game-over {
    color: red;
    font-size: 24px;
    margin-top: 10px;
    display: none;
  }
  #restart-btn {
    background: #333;
    color: white;
    border: 1px solid lime;
    padding: 8px 15px;
    margin-top: 10px;
    cursor: pointer;
    border-radius: 5px;
    display: none;
  }
  #restart-btn:hover {
    background: lime;
    color: black;
  }
</style>

<div class="game-container">
  <div id="score">Score: 0</div>
  <canvas id="snake-game" width="400" height="400"></canvas>
  <div id="game-over">GAME OVER!</div>
  <button id="restart-btn">Play Again</button>
</div>

<script>
  const canvas = document.getElementById('snake-game');
  const ctx = canvas.getContext('2d');
  const scoreElement = document.getElementById('score');
  const gameOverElement = document.getElementById('game-over');
  const restartBtn = document.getElementById('restart-btn');
  
  const grid = 20;
  let count = 0;
  let snake = [{x: 200, y: 200}];
  let dx = grid;
  let dy = 0;
  let apple = {x: 0, y: 0};
  let score = 0;
  let gameRunning = true;
  let speed = 6; // Initial speed (lower is faster)
  let gameLoopId;

  // Place apple at random position
  function placeApple() {
    apple.x = Math.floor(Math.random() * (canvas.width/grid)) * grid;
    apple.y = Math.floor(Math.random() * (canvas.height/grid)) * grid;
    
    // Make sure apple doesn't spawn on snake
    while (snake.some(segment => segment.x === apple.x && segment.y === apple.y)) {
      apple.x = Math.floor(Math.random() * (canvas.width/grid)) * grid;
      apple.y = Math.floor(Math.random() * (canvas.height/grid)) * grid;
    }
  }

  // Initial apple placement
  placeApple();

  function drawGame() {
    // Clear canvas
    ctx.clearRect(0, 0, canvas.width, canvas.height);
    
    // Draw border
    ctx.strokeStyle = '#333';
    ctx.lineWidth = 2;
    ctx.strokeRect(0, 0, canvas.width, canvas.height);
    
    // Draw apple (with gradient for better look)
    const appleGradient = ctx.createRadialGradient(
      apple.x + grid/2, apple.y + grid/2, 0,
      apple.x + grid/2, apple.y + grid/2, grid/2
    );
    appleGradient.addColorStop(0, '#ff0000');
    appleGradient.addColorStop(1, '#990000');
    ctx.fillStyle = appleGradient;
    ctx.beginPath();
    ctx.arc(apple.x + grid/2, apple.y + grid/2, grid/2 - 1, 0, Math.PI * 2);
    ctx.fill();
    
    // Draw snake
    snake.forEach((segment, index) => {
      // Head is different color
      if (index === 0) {
        ctx.fillStyle = '#00ff00';
      } else {
        // Body with gradient from head to tail
        const intensity = 150 + Math.floor(105 * (1 - index/snake.length));
        ctx.fillStyle = `rgb(0, ${intensity}, 0)`;
      }
      ctx.fillRect(segment.x, segment.y, grid-1, grid-1);
      
      // Add eyes to the head
      if (index === 0) {
        ctx.fillStyle = 'white';
        // Eyes point in direction snake is moving
        if (dx > 0) { // Right
          ctx.fillRect(segment.x + 12, segment.y + 3, 4, 4);
          ctx.fillRect(segment.x + 12, segment.y + 12, 4, 4);
        } else if (dx < 0) { // Left
          ctx.fillRect(segment.x + 3, segment.y + 3, 4, 4);
          ctx.fillRect(segment.x + 3, segment.y + 12, 4, 4);
        } else if (dy > 0) { // Down
          ctx.fillRect(segment.x + 3, segment.y + 12, 4, 4);
          ctx.fillRect(segment.x + 12, segment.y + 12, 4, 4);
        } else { // Up
          ctx.fillRect(segment.x + 3, segment.y + 3, 4, 4);
          ctx.fillRect(segment.x + 12, segment.y + 3, 4, 4);
        }
      }
    });
  }

  function updateGame() {
    if (!gameRunning) return;
    
    if (++count >= speed) {
      count = 0;
      
      // Move snake - wrap around if hitting walls
      let headX = snake[0].x + dx;
      let headY = snake[0].y + dy;
      
      // Wrap around horizontally
      if (headX >= canvas.width) headX = 0;
      else if (headX < 0) headX = canvas.width - grid;
      
      // Wrap around vertically
      if (headY >= canvas.height) headY = 0;
      else if (headY < 0) headY = canvas.height - grid;
      
      const head = {x: headX, y: headY};
      snake.unshift(head);
      
      // Check if snake ate apple
      if (head.x === apple.x && head.y === apple.y) {
        score += 10;
        scoreElement.textContent = `Score: ${score}`;
        placeApple();
        
        // Increase speed every 3 apples
        if (score % 30 === 0 && speed > 3) {
          speed -= 1;
        }
      } else {
        snake.pop();
      }
      
      // Check for self-collision only
      if (snake.slice(1).some(seg => seg.x === head.x && seg.y === head.y)) {
        endGame();
      }
    }
  }

  function endGame() {
    gameRunning = false;
    gameOverElement.style.display = 'block';
    restartBtn.style.display = 'inline-block';
    cancelAnimationFrame(gameLoopId);
  }

  function resetGame() {
    snake = [{x: 200, y: 200}];
    dx = grid;
    dy = 0;
    score = 0;
    speed = 6;
    scoreElement.textContent = `Score: ${score}`;
    placeApple();
    gameOverElement.style.display = 'none';
    restartBtn.style.display = 'none';
    gameRunning = true;
    gameLoopId = requestAnimationFrame(mainLoop);
  }

  function mainLoop() {
    updateGame();
    drawGame();
    if (gameRunning) {
      gameLoopId = requestAnimationFrame(mainLoop);
    }
  }

  // Keyboard controls
  document.addEventListener('keydown', e => {
    if (!gameRunning) return;
    
    // Prevent reverse direction (can't go right if moving left)
    if (e.key === 'ArrowLeft' && dx === 0) { dx = -grid; dy = 0; }
    else if (e.key === 'ArrowUp' && dy === 0) { dx = 0; dy = -grid; }
    else if (e.key === 'ArrowRight' && dx === 0) { dx = grid; dy = 0; }
    else if (e.key === 'ArrowDown' && dy === 0) { dx = 0; dy = grid; }
  });

  // Touch controls for mobile
  let touchStartX = 0;
  let touchStartY = 0;
  
  canvas.addEventListener('touchstart', e => {
    if (!gameRunning) return;
    touchStartX = e.touches[0].clientX;
    touchStartY = e.touches[0].clientY;
  }, false);
  
  canvas.addEventListener('touchmove', e => {
    if (!gameRunning) return;
    e.preventDefault();
    
    const touchEndX = e.touches[0].clientX;
    const touchEndY = e.touches[0].clientY;
    const dxTouch = touchEndX - touchStartX;
    const dyTouch = touchEndY - touchStartY;
    
    // Determine swipe direction
    if (Math.abs(dxTouch) > Math.abs(dyTouch)) {
      if (dxTouch > 0 && dx === 0) { dx = grid; dy = 0; } // Right
      else if (dxTouch < 0 && dx === 0) { dx = -grid; dy = 0; } // Left
    } else {
      if (dyTouch > 0 && dy === 0) { dx = 0; dy = grid; } // Down
      else if (dyTouch < 0 && dy === 0) { dx = 0; dy = -grid; } // Up
    }
  }, false);

  // Restart button
  restartBtn.addEventListener('click', resetGame);

  // Start the game
  gameLoopId = requestAnimationFrame(mainLoop);
</script>
