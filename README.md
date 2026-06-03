<!DOCTYPE html>
<html lang="id">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no, viewport-fit=cover">
    <title>IQ Parkour: Misi Otak & Lompatan Cerdas</title>
    <style>
        * {
            user-select: none;
            -webkit-tap-highlight-color: transparent;
        }

        body {
            margin: 0;
            min-height: 100vh;
            background: linear-gradient(145deg, #0a2f3a 0%, #05161c 100%);
            display: flex;
            justify-content: center;
            align-items: center;
            font-family: 'Segoe UI', 'Poppins', system-ui, -apple-system, 'Roboto', sans-serif;
            touch-action: manipulation;
        }

        /* Container utama responsif */
        .game-container {
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            padding: 12px 8px;
            width: 100%;
            max-width: 500px;
            margin: 0 auto;
        }

        canvas {
            display: block;
            margin: 0 auto;
            border-radius: 28px;
            box-shadow: 0 20px 35px rgba(0, 0, 0, 0.5), inset 0 1px 3px rgba(255,255,255,0.2);
            background: #6ec3b0;
            cursor: pointer;
            width: 100%;
            height: auto;
        }

        /* Panel informasi atas */
        .info-panel {
            background: rgba(0, 0, 0, 0.7);
            backdrop-filter: blur(10px);
            border-radius: 60px;
            padding: 10px 18px;
            margin-bottom: 15px;
            width: 100%;
            display: flex;
            justify-content: space-between;
            align-items: center;
            flex-wrap: wrap;
            gap: 12px;
            box-sizing: border-box;
            border: 1px solid rgba(255,215,0,0.5);
        }

        .stat {
            background: #1e2a32;
            padding: 6px 14px;
            border-radius: 40px;
            color: #f9e45b;
            font-weight: bold;
            font-size: 1rem;
            display: flex;
            align-items: center;
            gap: 8px;
            box-shadow: inset 0 1px 2px #2e4a5a, 0 3px 5px rgba(0,0,0,0.2);
        }

        .stat span:first-child {
            font-size: 1.2rem;
        }

        .level-badge {
            background: #8b5cf6;
            color: white;
            padding: 6px 16px;
            border-radius: 40px;
            font-weight: bold;
            letter-spacing: 1px;
        }

        .control-buttons {
            display: flex;
            gap: 20px;
            margin-top: 18px;
            margin-bottom: 8px;
            justify-content: center;
        }

        .ctrl-btn {
            background: #2c2e3a;
            border: none;
            font-size: 1.8rem;
            font-weight: bold;
            color: white;
            padding: 12px 28px;
            border-radius: 60px;
            box-shadow: 0 6px 0 #12141c;
            transition: 0.08s linear;
            touch-action: manipulation;
            cursor: pointer;
            font-family: monospace;
        }

        .ctrl-btn:active {
            transform: translateY(3px);
            box-shadow: 0 2px 0 #12141c;
        }

        .restart-btn {
            background: #e07c3c;
            box-shadow: 0 6px 0 #8b3c1a;
        }

        .info-text {
            font-size: 0.75rem;
            text-align: center;
            color: #bbe4dc;
            margin-top: 12px;
            background: #00000066;
            padding: 5px 12px;
            border-radius: 40px;
        }

        @media (max-width: 550px) {
            .stat { font-size: 0.8rem; padding: 4px 12px; }
            .ctrl-btn { padding: 8px 20px; font-size: 1.4rem; }
            .level-badge { font-size: 0.8rem; }
        }
    </style>
</head>
<body>
<div class="game-container">
    <div class="info-panel">
        <div class="stat">🧠 <span id="iqScore">0</span> IQ</div>
        <div class="level-badge" id="levelDisplay">LEVEL 1</div>
        <div class="stat">🎯 <span id="missionText">Koin: 0/3</span></div>
    </div>

    <canvas id="gameCanvas" width="450" height="500"></canvas>

    <div class="control-buttons">
        <button class="ctrl-btn" id="jumpBtn">🦘 LONCAT</button>
        <button class="ctrl-btn restart-btn" id="restartBtn">🔄 RESTART</button>
    </div>
    <div class="info-text">
        💡 TIPS: Kumpulkan koin sesuai misi → naik level! +IQ setiap misi selesai.<br>
        📱 Tap layar / tombol lompat. Hindari jurang & rintangan!
    </div>
</div>

<script>
    (function(){
        // ---------- CANVAS ----------
        const canvas = document.getElementById('gameCanvas');
        const ctx = canvas.getContext('2d');

        // ukuran fixed untuk gameplay (skala responsif via CSS)
        const CANVAS_W = 450;
        const CANVAS_H = 500;
        canvas.width = CANVAS_W;
        canvas.height = CANVAS_H;

        // ---------- GAME STATE ----------
        let gameRunning = true;
        let score = 0;          // IQ poin
        let level = 1;
        let currentMission = { type: "collect", target: 3, current: 0 }; // kumpulkan koin
        let missionCompletedFlag = false;

        // player
        const PLAYER_WIDTH = 28;
        const PLAYER_HEIGHT = 28;
        let player = {
            x: 70,
            y: 0,
            vy: 0,
            width: PLAYER_WIDTH,
            height: PLAYER_HEIGHT,
            grounded: true
        };
        const GRAVITY = 0.8;
        const JUMP_POWER = -11;

        // platform & rintangan
        let platforms = [];
        let coins = [];
        let obstacles = [];   // rintangan (menyentuh = game over)

        // camera offset (horizontal scroll)
        let cameraX = 0;
        const WORLD_WIDTH = 2800;   // dunia lebih panjang untuk level progresif

        // generasi awal
        function generateWorld() {
            platforms = [];
            coins = [];
            obstacles = [];

            // platform dasar awal (tanah)
            platforms.push({ x: 0, y: CANVAS_H - 45, w: 350, h: 20 });
            platforms.push({ x: 380, y: CANVAS_H - 80, w: 90, h: 20 });
            platforms.push({ x: 520, y: CANVAS_H - 130, w: 80, h: 20 });
            platforms.push({ x: 660, y: CANVAS_H - 90, w: 70, h: 20 });
            platforms.push({ x: 800, y: CANVAS_H - 160, w: 100, h: 20 });
            platforms.push({ x: 960, y: CANVAS_H - 120, w: 70, h: 20 });
            platforms.push({ x: 1100, y: CANVAS_H - 190, w: 90, h: 20 });
            platforms.push({ x: 1280, y: CANVAS_H - 100, w: 110, h: 20 });
            platforms.push({ x: 1460, y: CANVAS_H - 170, w: 80, h: 20 });
            platforms.push({ x: 1620, y: CANVAS_H - 210, w: 100, h: 20 });
            platforms.push({ x: 1800, y: CANVAS_H - 140, w: 85, h: 20 });
            platforms.push({ x: 1970, y: CANVAS_H - 80, w: 120, h: 20 });
            platforms.push({ x: 2150, y: CANVAS_H - 180, w: 100, h: 20 });
            platforms.push({ x: 2350, y: CANVAS_H - 110, w: 130, h: 20 });
            platforms.push({ x: 2550, y: CANVAS_H - 50, w: 200, h: 20 }); // finish land

            // SESUAIKAN dengan level : obstacle dan koin makin rumit berdasarkan level
            function addCoinsAndObstaclesByLevel() {
                // standard coins di berbagai platform
                const coinSpots = [
                    { x: 100, y: CANVAS_H - 70 }, { x: 280, y: CANVAS_H - 70 },
                    { x: 420, y: CANVAS_H - 105 }, { x: 560, y: CANVAS_H - 155 },
                    { x: 700, y: CANVAS_H - 115 }, { x: 850, y: CANVAS_H - 185 },
                    { x: 1000, y: CANVAS_H - 145 }, { x: 1150, y: CANVAS_H - 215 },
                    { x: 1320, y: CANVAS_H - 125 }, { x: 1500, y: CANVAS_H - 195 },
                    { x: 1660, y: CANVAS_H - 235 }, { x: 1840, y: CANVAS_H - 165 },
                    { x: 2010, y: CANVAS_H - 105 }, { x: 2200, y: CANVAS_H - 205 },
                    { x: 2400, y: CANVAS_H - 135 }, { x: 2600, y: CANVAS_H - 75 }
                ];
                for(let spot of coinSpots) {
                    coins.push({ x: spot.x, y: spot.y, w: 12, h: 12, collected: false });
                }

                // obstacles: level mempengaruhi frekuensi dan pola
                let obstacleList = [];
                if(level >= 2) {
                    obstacleList.push({ x: 470, y: CANVAS_H - 85, w: 18, h: 18 });
                    obstacleList.push({ x: 930, y: CANVAS_H - 145, w: 20, h: 20 });
                }
                if(level >= 3) {
                    obstacleList.push({ x: 1210, y: CANVAS_H - 200, w: 22, h: 22 });
                    obstacleList.push({ x: 1530, y: CANVAS_H - 180, w: 20, h: 20 });
                    obstacleList.push({ x: 1890, y: CANVAS_H - 150, w: 25, h: 25 });
                }
                if(level >= 4) {
                    obstacleList.push({ x: 2050, y: CANVAS_H - 110, w: 20, h: 20 });
                    obstacleList.push({ x: 2270, y: CANVAS_H - 190, w: 24, h: 24 });
                    obstacleList.push({ x: 2460, y: CANVAS_H - 120, w: 22, h: 22 });
                }
                if(level >= 5) {
                    obstacleList.push({ x: 2650, y: CANVAS_H - 65, w: 28, h: 28 });
                    obstacleList.push({ x: 2720, y: CANVAS_H - 65, w: 28, h: 28 });
                }
                for(let obs of obstacleList) {
                    obstacles.push(obs);
                }
            }

            addCoinsAndObstaclesByLevel();

            // Tambah misi dinamis: target = minimal 3 + level/2 (makin tinggi level butuh koin lebih)
            let missionTarget = Math.min(12, 3 + Math.floor(level / 2));
            currentMission = {
                type: "collect",
                target: missionTarget,
                current: 0
            };
            missionCompletedFlag = false;
            document.getElementById("missionText").innerText = `Koin: 0/${missionTarget}`;
        }

        // reset permainan penuh
        function fullReset() {
            gameRunning = true;
            score = 0;
            level = 1;
            updateUI();
            generateWorld();        // generate sesuai level=1
            player.x = 70;
            player.y = 0;
            player.vy = 0;
            player.grounded = true;
            cameraX = 0;
            // sync mission display
            document.getElementById("missionText").innerText = `Koin: 0/${currentMission.target}`;
            document.getElementById("iqScore").innerText = score;
            document.getElementById("levelDisplay").innerText = `LEVEL ${level}`;
            missionCompletedFlag = false;
            // pastikan posisi y player di platform
            adjustPlayerToGround();
        }

        // level naik, +IQ, dan reset dunia dengan level baru
        function levelUp() {
            level++;
            // tambah IQ besar setiap naik level (prestasi kognitif)
            let iqGain = 15 + level * 2;
            score += iqGain;
            updateUI();
            // reset dunia berdasarkan level baru
            generateWorld();     // otomatis mission target baru dan rintangan lebih sulit
            player.x = 70;
            player.vy = 0;
            cameraX = 0;
            adjustPlayerToGround();
            gameRunning = true;
            missionCompletedFlag = false;
            // update tampilan misi
            document.getElementById("missionText").innerText = `Koin: 0/${currentMission.target}`;
            document.getElementById("levelDisplay").innerText = `LEVEL ${level}`;
            document.getElementById("iqScore").innerText = score;
        }

        function updateUI() {
            document.getElementById("iqScore").innerText = Math.floor(score);
            document.getElementById("levelDisplay").innerText = `LEVEL ${level}`;
        }

        // menyesuaikan player di platform terdekat (spawn aman)
        function adjustPlayerToGround() {
            let found = false;
            for(let plat of platforms) {
                if(player.x + PLAYER_WIDTH > plat.x && player.x < plat.x + plat.w) {
                    player.y = plat.y - PLAYER_HEIGHT;
                    player.vy = 0;
                    player.grounded = true;
                    found = true;
                    break;
                }
            }
            if(!found && platforms.length>0) {
                player.y = platforms[0].y - PLAYER_HEIGHT;
                player.grounded = true;
            } else if(!found) {
                player.y = CANVAS_H - PLAYER_HEIGHT - 10;
            }
        }

        // fisika dan collision
        function updateGame() {
            if(!gameRunning) return;

            // gravitasi
            player.vy += GRAVITY;
            player.y += player.vy;

            // collision platform
            player.grounded = false;
            for(let plat of platforms) {
                if(player.x + PLAYER_WIDTH > plat.x && player.x < plat.x + plat.w) {
                    if(player.vy >= 0 && player.y + PLAYER_HEIGHT <= plat.y + 15 && player.y + PLAYER_HEIGHT + player.vy >= plat.y) {
                        player.y = plat.y - PLAYER_HEIGHT;
                        player.vy = 0;
                        player.grounded = true;
                    }
                }
            }

            // batas bawah jatuh
            if(player.y + PLAYER_HEIGHT > CANVAS_H) {
                gameRunning = false;
                return;
            }

            // batas atas
            if(player.y < 0) player.y = 0;

            // scroll kamera berdasarkan player
            let targetCam = player.x + PLAYER_WIDTH/2 - CANVAS_W/2;
            targetCam = Math.min(Math.max(targetCam, 0), WORLD_WIDTH - CANVAS_W);
            cameraX = targetCam;

            // collect koin
            for(let i=0; i<coins.length; i++) {
                let c = coins[i];
                if(!c.collected && player.x < c.x + c.w && player.x+PLAYER_WIDTH > c.x && player.y < c.y + c.h && player.y+PLAYER_HEIGHT > c.y) {
                    c.collected = true;
                    currentMission.current++;
                    document.getElementById("missionText").innerText = `Koin: ${currentMission.current}/${currentMission.target}`;
                    // tambah sedikit IQ saat ambil koin (stimulus)
                    score += 2;
                    updateUI();

                    // cek misi selesai
                    if(currentMission.current >= currentMission.target && !missionCompletedFlag) {
                        missionCompletedFlag = true;
                        // naik level otomatis!
                        levelUp();
                    }
                }
            }

            // cek rintangan (game over)
            for(let obs of obstacles) {
                if(player.x < obs.x + obs.w && player.x+PLAYER_WIDTH > obs.x && player.y < obs.y+obs.h && player.y+PLAYER_HEIGHT > obs.y) {
                    gameRunning = false;
                    break;
                }
            }

            // bonus cek jika mencapai akhir dunia (IQ bonus)
            if(player.x + PLAYER_WIDTH > WORLD_WIDTH - 50 && gameRunning){
                // menyelesaikan world ekstra hadiah IQ
                score += 25;
                updateUI();
                levelUp(); // naik level extra
            }
        }

        // GAMBAR SEMUA
        function draw() {
            ctx.clearRect(0, 0, CANVAS_W, CANVAS_H);
            // background gradasi langit
            let grad = ctx.createLinearGradient(0,0,0,CANVAS_H);
            grad.addColorStop(0,"#c1e3fe");
            grad.addColorStop(1,"#8fc1b0");
            ctx.fillStyle = grad;
            ctx.fillRect(0,0,CANVAS_W,CANVAS_H);

            // gambar platform
            for(let plat of platforms) {
                let drawX = plat.x - cameraX;
                if(drawX + plat.w > 0 && drawX < CANVAS_W) {
                    ctx.fillStyle = "#9b6a4c";
                    ctx.shadowBlur=0;
                    ctx.fillRect(drawX, plat.y, plat.w, plat.h);
                    ctx.fillStyle = "#c28a5a";
                    ctx.fillRect(drawX, plat.y-4, plat.w, 6);
                    ctx.fillStyle = "#6b3e1c";
                    ctx.fillRect(drawX, plat.y+plat.h-3, plat.w, 4);
                }
            }

            // koin
            for(let c of coins) {
                if(!c.collected) {
                    let drawX = c.x - cameraX;
                    if(drawX + c.w > 0 && drawX < CANVAS_W) {
                        ctx.fillStyle = "#f7d44a";
                        ctx.beginPath();
                        ctx.ellipse(drawX+c.w/2, c.y+c.h/2, c.w/2, c.h/2, 0, 0, Math.PI*2);
                        ctx.fill();
                        ctx.fillStyle = "#e5b800";
                        ctx.beginPath();
                        ctx.ellipse(drawX+c.w/2, c.y+c.h/2, c.w/3, c.h/3, 0, 0, Math.PI*2);
                        ctx.fill();
                        ctx.fillStyle = "white";
                        ctx.font = "bold 8px monospace";
                        ctx.fillText("⚡", drawX+2, c.y+9);
                    }
                }
            }

            // rintangan (batu berbahaya)
            for(let obs of obstacles) {
                let drawX = obs.x - cameraX;
                if(drawX + obs.w > 0 && drawX < CANVAS_W) {
                    ctx.fillStyle = "#4a2a1e";
                    ctx.shadowBlur = 3;
                    ctx.fillRect(drawX, obs.y, obs.w, obs.h);
                    ctx.fillStyle = "#2c1a12";
                    ctx.fillRect(drawX+3, obs.y-2, obs.w-6, 4);
                    ctx.fillStyle = "#ff6666";
                    ctx.font = "bold 12px monospace";
                    ctx.fillText("⚠", drawX+4, obs.y+obs.h-4);
                    ctx.shadowBlur = 0;
                }
            }

            // karakter Parkour (pintar)
            let playerDrawX = player.x - cameraX;
            ctx.shadowBlur = 0;
            // body
            ctx.fillStyle = "#3c8f7a";
            ctx.beginPath();
            ctx.roundRect(playerDrawX, player.y, PLAYER_WIDTH, PLAYER_HEIGHT, 8);
            ctx.fill();
            ctx.fillStyle = "#f5cd7a";
            ctx.beginPath();
            ctx.arc(playerDrawX+PLAYER_WIDTH*0.7, player.y+8, 6, 0, Math.PI*2);
            ctx.fill();
            ctx.fillStyle = "#1f2e2a";
            ctx.beginPath();
            ctx.arc(playerDrawX+PLAYER_WIDTH*0.7, player.y+6, 2, 0, Math.PI*2);
            ctx.fill();
            ctx.fillStyle = "white";
            ctx.font = "bold 15px monospace";
            ctx.fillText("🧠", playerDrawX+5, player.y+22);
            // topi
            ctx.fillStyle = "#e07c3c";
            ctx.fillRect(playerDrawX+2, player.y-5, 24, 7);
            // ransel
            ctx.fillStyle = "#c46b3a";
            ctx.fillRect(playerDrawX+PLAYER_WIDTH-8, player.y+12, 8, 12);

            // tampilkan pesan game over
            if(!gameRunning) {
                ctx.font = "bold 24px 'Segoe UI'";
                ctx.fillStyle = "#c02828";
                ctx.shadowBlur = 6;
                ctx.fillText("GAME OVER", CANVAS_W/2-90, CANVAS_H/2-40);
                ctx.font = "16px monospace";
                ctx.fillStyle = "white";
                ctx.fillText("Tekan RESTART", CANVAS_W/2-65, CANVAS_H/2+20);
                ctx.shadowBlur = 0;
            }

            // Petunjuk misi level
            ctx.font = "bold 12px monospace";
            ctx.fillStyle = "#111";
            ctx.fillText(`📜 Misi: kumpulkan ${currentMission.target} koin`, 12, 35);
            ctx.fillStyle = "#f3c26b";
            ctx.fillText(`⭐ IQ: ${Math.floor(score)}  |  level ${level}`, 12, 58);
        }

        // helper roundRect
        if (!CanvasRenderingContext2D.prototype.roundRect) {
        
