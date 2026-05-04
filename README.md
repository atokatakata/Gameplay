<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Mobile Survivors</title>
    <style>
        body {
            margin: 0;
            overflow: hidden;
            background-color: #2c3e50;
            font-family: sans-serif;
            touch-action: none; /* ブラウザ標準のスクロールなどを無効化 */
        }
        #canvas { display: block; background-color: #34495e; }
        #ui {
            position: absolute;
            top: 10px;
            left: 10px;
            pointer-events: none;
            color: white;
        }
        .hp-bar-container {
            width: 150px;
            height: 15px;
            background-color: #ccc;
            border-radius: 10px;
            overflow: hidden;
            margin-bottom: 5px;
        }
        #hp-bar { width: 100%; height: 100%; background-color: #2ecc71; }
        #gameover-screen {
            display: none;
            position: absolute;
            top: 0; left: 0; width: 100%; height: 100%;
            background-color: rgba(0, 0, 0, 0.8);
            flex-direction: column; align-items: center; justify-content: center;
            color: white; z-index: 10;
        }
        #gameover-screen.active { display: flex; }
        #restart-btn {
            background-color: #e67e22; color: white; border: none;
            padding: 15px 30px; font-size: 1.2em; border-radius: 30px; margin-top: 20px;
        }
        /* スティックの見た目 */
        #joystick-base {
            position: absolute;
            bottom: 50px;
            left: 50px;
            width: 100px;
            height: 100px;
            background: rgba(255, 255, 255, 0.2);
            border-radius: 50%;
            display: none; /* タッチ中のみ表示 */
            pointer-events: none;
        }
        #joystick-stick {
            position: absolute;
            width: 40px;
            height: 40px;
            background: rgba(255, 255, 255, 0.5);
            border-radius: 50%;
            top: 30px;
            left: 30px;
        }
    </style>
</head>
<body>

<canvas id="canvas"></canvas>

<div id="joystick-base">
    <div id="joystick-stick"></div>
</div>

<div id="ui">
    <div class="hp-bar-container"><div id="hp-bar"></div></div>
    <div style="font-size: 0.9em;">Lv: <span id="level-val">1</span> | Score: <span id="score-val">0</span></div>
</div>

<div id="gameover-screen">
    <h1>GAME OVER</h1>
    <p>Score: <span id="final-score">0</span></p>
    <button id="restart-btn" onclick="location.reload()">RETRY</button>
</div>

<script>
    const canvas = document.getElementById('canvas');
    const ctx = canvas.getContext('2d');
    const hpBar = document.getElementById('hp-bar');
    const joystickBase = document.getElementById('joystick-base');
    const joystickStick = document.getElementById('joystick-stick');

    let width, height, player, enemies = [], projectiles = [], exps = [];
    let isGameOver = false, score = 0, level = 1, exp = 0, expToNext = 5;
    let frameCount = 0;

    // スティックの状態
    let touchX = 0, touchY = 0, dragX = 0, dragY = 0, isDragging = false;

    function resize() {
        width = canvas.width = window.innerWidth;
        height = canvas.height = window.innerHeight;
    }
    window.addEventListener('resize', resize);
    resize();

    class Player {
        constructor() {
            this.x = width / 2; this.y = height / 2;
            this.r = 15; this.speed = 3; this.hp = 20; this.maxHp = 20;
            this.lastAttack = 0; this.attackSpeed = 800;
        }
        update() {
            if (isDragging) {
                const dist = Math.hypot(dragX, dragY);
                if (dist > 0) {
                    this.x += (dragX / dist) * this.speed;
                    this.y += (dragY / dist) * this.speed;
                }
            }
            this.x = Math.max(this.r, Math.min(width - this.r, this.x));
            this.y = Math.max(this.r, Math.min(height - this.r, this.y));
        }
        draw() {
            ctx.fillStyle = '#3498db';
            ctx.beginPath(); ctx.arc(this.x, this.y, this.r, 0, Math.PI*2); ctx.fill();
        }
    }

    class Enemy {
        constructor() {
            this.r = 12; this.hp = 1 + Math.floor(level/2);
            const side = Math.floor(Math.random() * 4);
            if(side===0) { this.x = Math.random()*width; this.y = -20; }
            if(side===1) { this.x = width+20; this.y = Math.random()*height; }
            if(side===2) { this.x = Math.random()*width; this.y = height+20; }
            if(side===3) { this.x = -20; this.y = Math.random()*height; }
            this.speed = 1 + Math.random();
        }
        update() {
            const angle = Math.atan2(player.y - this.y, player.x - this.x);
            this.x += Math.cos(angle) * this.speed;
            this.y += Math.sin(angle) * this.speed;
        }
        draw() {
            ctx.fillStyle = '#e74c3c';
            ctx.beginPath(); ctx.arc(this.x, this.y, this.r, 0, Math.PI*2); ctx.fill();
        }
    }

    class Projectile {
        constructor(tx, ty) {
            this.x = player.x; this.y = player.y; this.r = 5;
            const angle = Math.atan2(ty - this.y, tx - this.x);
            this.vx = Math.cos(angle) * 7; this.vy = Math.sin(angle) * 7;
        }
        update() { this.x += this.vx; this.y += this.vy; }
        draw() {
            ctx.fillStyle = '#f1c40f';
            ctx.beginPath(); ctx.arc(this.x, this.y, this.r, 0, Math.PI*2); ctx.fill();
        }
    }

    player = new Player();

    // タッチイベント
    window.addEventListener('touchstart', e => {
        isDragging = true;
        touchX = e.touches[0].clientX;
        touchY = e.touches[0].clientY;
        joystickBase.style.display = 'block';
        joystickBase.style.left = (touchX - 50) + 'px';
        joystickBase.style.top = (touchY - 50) + 'px';
    });

    window.addEventListener('touchmove', e => {
        if (!isDragging) return;
        dragX = e.touches[0].clientX - touchX;
        dragY = e.touches[0].clientY - touchY;
        const limit = 40;
        const dist = Math.hypot(dragX, dragY);
        if (dist > limit) {
            dragX = (dragX / dist) * limit;
            dragY = (dragY / dist) * limit;
        }
        joystickStick.style.transform = `translate(${dragX}px, ${dragY}px)`;
    });

    window.addEventListener('touchend', () => {
        isDragging = false;
        dragX = 0; dragY = 0;
        joystickBase.style.display = 'none';
    });

    function gameLoop(time) {
        if (isGameOver) return;
        ctx.clearRect(0, 0, width, height);

        player.update();
        player.draw();

        if (time - player.lastAttack > player.attackSpeed && enemies.length > 0) {
            let target = enemies[0];
            projectiles.push(new Projectile(target.x, target.y));
            player.lastAttack = time;
        }

        projectiles.forEach((p, i) => {
            p.update(); p.draw();
            if(p.x<0||p.x>width||p.y<0||p.y>height) projectiles.splice(i, 1);
        });

        enemies.forEach((e, i) => {
            e.update(); e.draw();
            if (Math.hypot(e.x - player.x, e.y - player.y) < e.r + player.r) {
                player.hp -= 0.1;
                if(player.hp <= 0) { isGameOver = true; document.getElementById('gameover-screen').classList.add('active'); document.getElementById('final-score').textContent = score; }
            }
            projectiles.forEach((p, pi) => {
                if (Math.hypot(e.x - p.x, e.y - p.y) < e.r + p.r) {
                    e.hp--; projectiles.splice(pi, 1);
                    if(e.hp<=0) { score += 10; enemies.splice(i, 1); exp++; }
                }
            });
        });

        if (exp >= expToNext) { level++; exp = 0; expToNext += 5; player.attackSpeed *= 0.9; player.hp = player.maxHp; }
        if (frameCount % 60 === 0) enemies.push(new Enemy());

        hpBar.style.width = (player.hp / player.maxHp * 100) + '%';
        document.getElementById('level-val').textContent = level;
        document.getElementById('score-val').textContent = score;

        frameCount++;
        requestAnimationFrame(gameLoop);
    }
    requestAnimationFrame(gameLoop);
</script>
</body>
</html>
