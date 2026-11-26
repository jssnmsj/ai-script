export default {
  async fetch(request, env, ctx) {
    const html = `<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Executor X Remote Spy x Ai</title>
  <style>
    body {
      margin: 0;
      padding: 20px;
      background-color: #000000;
      color: #ffffff;
      font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif;
      min-height: 100vh;
      display: flex;
      justify-content: center;
      align-items: center;
      overflow-x: hidden;
    }
    canvas {
      position: fixed;
      top: 0;
      left: 0;
      width: 100%;
      height: 100%;
      pointer-events: none;
      z-index: 1;
    }
    .container {
      position: relative;
      z-index: 2;
      max-width: 600px;
      width: 100%;
      background: rgba(20, 20, 20, 0.85);
      border: 1px solid #333333;
      border-radius: 12px;
      padding: 32px;
      text-align: center;
      box-shadow: 0 8px 32px rgba(0, 0, 0, 0.8);
    }
    h1 {
      font-size: 1.8rem;
      margin-top: 0;
      margin-bottom: 24px;
      color: #ffffff;
    }
    h2 {
      font-size: 1.25rem;
      margin-top: 24px;
      margin-bottom: 10px;
      color: #e0e0e0;
    }
    p {
      font-size: 1rem;
      line-height: 1.6;
      color: #cccccc;
      margin: 0 auto 16px;
    }
    button {
      margin-top: 10px;
      padding: 12px 28px;
      font-size: 1rem;
      font-weight: 600;
      color: #ffffff;
      background-color: #2563eb;
      border: none;
      border-radius: 8px;
      cursor: pointer;
      transition: background-color 0.2s;
    }
    button:hover {
      background-color: #1d4ed8;
    }
    #status-box {
      display: none;
      margin-top: 20px;
      padding: 16px;
      background-color: #18181b;
      border: 1px solid #3f3f46;
      border-radius: 8px;
      font-size: 1rem;
      line-height: 1.5;
    }
  </style>
</head>
<body>
  <canvas id="snow"></canvas>

  <div class="container">
    <h1>Welcome To Executor X Remote Spy x Ai</h1>

    <h2>How Dose It Work?</h2>
    <p>Well Click Find And Once U Found Any Remote Go Ai Or Exec And code it or ask ai "Make It Loop This Remote And it Will Do Jobs For You".</p>

    <h2>How Do I Tell If it Online / Offline</h2>
    <button onclick="showStatus()">Script</button>

    <div id="status-box">
      <strong>Executor X Remote:</strong> 🟠<br>
      Maintence
    </div>
  </div>

  <script>
    function showStatus() {
      const box = document.getElementById('status-box');
      box.style.display = box.style.display === 'block' ? 'none' : 'block';
    }

    // Snow effect animation
    const canvas = document.getElementById('snow');
    const ctx = canvas.getContext('2d');
    let width = canvas.width = window.innerWidth;
    let height = canvas.height = window.innerHeight;

    window.addEventListener('resize', () => {
      width = canvas.width = window.innerWidth;
      height = canvas.height = window.innerHeight;
    });

    const flakes = Array.from({ length: 100 }, () => ({
      x: Math.random() * width,
      y: Math.random() * height,
      r: Math.random() * 3 + 1,
      speed: Math.random() * 1.5 + 0.5,
      drift: Math.random() * 0.6 - 0.3
    }));

    function drawSnow() {
      ctx.clearRect(0, 0, width, height);
      ctx.fillStyle = 'rgba(255, 255, 255, 0.8)';
      ctx.beginPath();
      for (const f of flakes) {
        ctx.moveTo(f.x, f.y);
        ctx.arc(f.x, f.y, f.r, 0, Math.PI * 2);
      }
      ctx.fill();

      for (const f of flakes) {
        f.y += f.speed;
        f.x += f.drift;
        if (f.y > height) {
          f.y = -5;
          f.x = Math.random() * width;
        }
        if (f.x > width) f.x = 0;
        if (f.x < 0) f.x = width;
      }
      requestAnimationFrame(drawSnow);
    }

    drawSnow();
  </script>
</body>
</html>`;

    return new Response(html, {
      headers: { 'content-type': 'text/html;charset=UTF-8' },
    });
  }
};
