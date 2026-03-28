<!DOCTYPE html>
<html lang="pt-br">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Lendário Dash 9.1 - Glow Edition</title>
    <style>
        :root { --accent: #00f2ff; --gold: #ffd700; --bg: #0a0a0c; --red: #ff0044; }
        * { box-sizing: border-box; margin: 0; padding: 0; font-family: 'Arial Black', sans-serif; user-select: none; }
        body { background: var(--bg); color: white; height: 100vh; display: flex; justify-content: center; align-items: center; overflow: hidden; }
        
        #game-container { 
            position: relative; width: 100%; max-width: 850px; aspect-ratio: 2 / 1; 
            background: #000; border: 4px solid var(--accent); box-shadow: 0 0 20px var(--accent); 
            overflow: hidden; border-radius: 15px; transition: border-color 0.5s, box-shadow 0.5s;
        }

        canvas { width: 100%; height: 100%; display: block; }
        .hud { position: absolute; top: 15px; left: 15px; z-index: 10; display: flex; gap: 10px; }
        .badge { background: rgba(0,0,0,0.7); padding: 8px 15px; border-radius: 50px; border: 1px solid var(--accent); font-size: 12px; }
        .overlay { position: absolute; inset: 0; z-index: 100; background: rgba(0,0,0,0.9); backdrop-filter: blur(10px); display: flex; flex-direction: column; justify-content: center; align-items: center; }
        .panel { background: rgba(255,255,255,0.05); padding: 30px; border-radius: 25px; border: 1px solid rgba(255,255,255,0.1); text-align: center; width: 90%; max-width: 600px; }
        .btn { padding: 15px 30px; border: none; border-radius: 15px; font-weight: 900; cursor: pointer; text-transform: uppercase; margin: 10px; transition: 0.2s; font-size: 14px; }
        .btn-play { background: var(--accent); color: #000; box-shadow: 0 0 20px rgba(0,242,255,0.5); width: 250px; }
        .btn-menu { background: #333; color: #fff; width: 200px; border: 3px solid transparent; }
        .btn-active { border: 3px solid var(--accent) !important; box-shadow: 0 0 15px var(--accent); color: var(--accent) !important; }
        .shop-grid { display: grid; grid-template-columns: repeat(5, 1fr); gap: 10px; margin: 10px 0; max-height: 250px; overflow-y: auto; padding: 10px; }
        .skin-card { background: rgba(255,255,255,0.05); padding: 10px; border-radius: 15px; border: 2px solid transparent; text-align: center; }
        .skin-card.active { border-color: var(--accent); background: rgba(0,242,255,0.1); }
        .btn-shop { padding: 8px; border: none; border-radius: 10px; font-weight: 900; font-size: 9px; width: 100%; cursor: pointer; text-transform: uppercase; margin-top: 5px; }
        .btn-buy { background: var(--gold); color: #000; }
        .btn-use { background: var(--accent); color: #000; }
        .btn-owned { background: #555; color: #eee; cursor: default; }
        .hidden { display: none !important; }
    </style>
</head>
<body>

    <div id="game-container">
        <div class="hud">
            <div class="badge">SCORE: <span id="score">0</span></div>
            <div class="badge" style="color:var(--gold)">💰 <span id="coins-hud">0</span></div>
        </div>

        <div id="main-menu" class="overlay">
            <div class="panel">
                <h1 style="color:var(--accent); font-size: 3rem; margin-bottom: 20px;">LENDÁRIO DASH</h1>
                <button class="btn btn-play" onclick="startGame()">JOGAR</button><br>
                <button class="btn btn-menu" onclick="abrirMenu('shop-menu')">LOJA DE SKINS</button><br>
                <button class="btn btn-menu" onclick="abrirMenu('config-menu')">CONFIGURAÇÕES</button>
            </div>
        </div>

        <div id="config-menu" class="overlay hidden">
            <div class="panel">
                <h2>CONFIGURAÇÕES</h2>
                <p style="font-size: 12px; margin-bottom: 15px; opacity: 0.6;">QUALIDADE GRÁFICA</p>
                <div style="display: flex; gap: 10px; margin-bottom: 20px; justify-content: center;">
                    <button class="btn btn-menu" style="width:100px" id="g-low" onclick="setGraphics('low')">BAIXO</button>
                    <button class="btn btn-menu" style="width:100px" id="g-med" onclick="setGraphics('med')">MÉDIO</button>
                    <button class="btn btn-menu" style="width:100px" id="g-pro" onclick="setGraphics('pro')">PRO</button>
                </div>
                <button class="btn btn-play" style="margin-top:20px" onclick="fecharMenus()">VOLTAR</button>
            </div>
        </div>

        <div id="shop-menu" class="overlay hidden">
            <div class="panel" style="max-width: 700px;">
                <h2 style="color:var(--gold); margin-bottom: 5px;">MERCADO DE SKINS</h2>
                <div id="shop-msg" style="height:20px; font-size:12px; margin-bottom:10px; opacity:0; transition:0.3s;"></div> 
                <div class="shop-grid" id="shop-items"></div>
                <button class="btn btn-play" onclick="fecharMenus()">VOLTAR</button>
            </div>
        </div>

        <canvas id="canvas" width="800" height="400"></canvas>
    </div>

<script>
    const canvas = document.getElementById('canvas');
    const ctx = canvas.getContext('2d');
    const gameContainer = document.getElementById('game-container');
    const audioCtx = new (window.AudioContext || window.webkitAudioContext)();
    
    let graphicsLvl = localStorage.getItem('L91_Graphics') || 'med';
    let coins = parseInt(localStorage.getItem('L91_Coins')) || 0; 
    let owned = JSON.parse(localStorage.getItem('L91_Owned')) || [0];
    let activeId = parseInt(localStorage.getItem('L91_Active')) || 0;

    const SKINS = [
        { id: 0, name: "Padrão", color: "#00f2ff", price: 0 },
        { id: 1, name: "LED CHROMA", color: "chroma", price: 100 },
        { id: 2, name: "Galaxy X", color: "#cc00ff", price: 50 },
        { id: 3, name: "Ouro Puro", color: "#ffd700", price: 150 }
    ];
    for(let i=4; i<30; i++) SKINS.push({ id: i, name: `Skin ${i+1}`, color: `hsl(${i*12}, 70%, 50%)`, price: 25+i });

    let gameActive = false, score = 0, lastSpawn = 0, particles = [];
    let obstacles = [], gameCoins = [];
    let player = { x: 120, y: 315, w: 35, h: 35, dy: 0, rot: 0, gravity: 0.65, jumpForce: -11, jumps: 0 };

    let lastTime = 0;
    const fps = 60;
    const interval = 1000 / fps;

    function sync() { 
        localStorage.setItem('L91_Coins', coins); 
        localStorage.setItem('L91_Owned', JSON.stringify(owned)); 
        localStorage.setItem('L91_Active', activeId); 
        document.getElementById('coins-hud').innerText = coins;
        
        const s = SKINS.find(x => x.id === activeId);
        if(s.color !== 'chroma') {
            gameContainer.style.borderColor = s.color;
            gameContainer.style.boxShadow = `0 0 20px ${s.color}`;
            document.querySelectorAll('.badge').forEach(b => b.style.borderColor = s.color);
        }
    }

    function playCoinSound() {
        if(audioCtx.state === 'suspended') audioCtx.resume();
        const osc = audioCtx.createOscillator(); const gain = audioCtx.createGain();
        osc.frequency.setValueAtTime(880, audioCtx.currentTime);
        gain.gain.setValueAtTime(0.05, audioCtx.currentTime);
        osc.connect(gain); gain.connect(audioCtx.destination);
        osc.start(); osc.stop(audioCtx.currentTime + 0.1);
    }

    function createCollectEffect(x, y) {
        for(let i=0; i<8; i++) particles.push({ x: x, y: y, vx: (Math.random()-0.5)*8, vy: (Math.random()-0.5)*8, life: 1.0, color: "#ffd700" });
    }

    function abrirMenu(id) { fecharMenus(); document.getElementById(id).classList.remove('hidden'); if(id === 'shop-menu') renderShop(); if(id === 'config-menu') atualizarBordasGraficos(); }
    function fecharMenus() { document.querySelectorAll('.overlay').forEach(o => o.classList.add('hidden')); if(!gameActive) document.getElementById('main-menu').classList.remove('hidden'); }
    function atualizarBordasGraficos() { document.querySelectorAll('.btn-menu').forEach(b => b.classList.remove('btn-active')); if(document.getElementById('g-' + graphicsLvl)) document.getElementById('g-' + graphicsLvl).classList.add('btn-active'); }
    function setGraphics(lvl) { graphicsLvl = lvl; localStorage.setItem('L91_Graphics', lvl); atualizarBordasGraficos(); }
    
    function renderShop() { 
        const grid = document.getElementById('shop-items'); grid.innerHTML = ''; 
        SKINS.forEach(s => { 
            const isOwned = owned.includes(s.id); const isActive = activeId === s.id; const card = document.createElement('div'); card.className = `skin-card ${isActive ? 'active' : ''}`; 
            let btnHtml = !isOwned ? `<button class="btn-shop btn-buy" onclick="buySkin(${s.id})">COMPRAR ${s.price}</button>` : (!isActive ? `<button class="btn-shop btn-use" onclick="useSkin(${s.id})">USAR</button>` : `<button class="btn-shop btn-owned">USANDO</button>`); 
            card.innerHTML = `<div style="width:30px; height:30px; background:${s.color === 'chroma' ? 'linear-gradient(red, blue)' : s.color}; margin: 0 auto 5px; border-radius:5px; border:1px solid #fff;"></div><div style="font-size:9px;">${s.name}</div>${btnHtml}`; 
            grid.appendChild(card); 
        }); 
    }

    function buySkin(id) { const s = SKINS.find(x => x.id === id); if(coins >= s.price) { coins -= s.price; owned.push(id); sync(); renderShop(); } }
    function useSkin(id) { activeId = id; sync(); renderShop(); }
    
    function startGame() { 
        if(audioCtx.state === 'suspended') audioCtx.resume(); 
        score = 0; obstacles = []; gameCoins = []; particles = []; 
        player.y = 315; player.dy = 0; player.rot = 0; player.jumps = 0; 
        fecharMenus(); gameActive = true; sync(); 
        lastTime = performance.now();
        requestAnimationFrame(update); 
    }

    function update(time) {
        if (!gameActive) return;
        requestAnimationFrame(update);

        const delta = time - lastTime;
        if (delta < interval) return; 
        lastTime = time - (delta % interval);

        const s = SKINS.find(x => x.id === activeId);
        let pColor = s.color === 'chroma' ? `hsl(${(time/5)%360}, 100%, 50%)` : s.color;
        if(s.color === 'chroma') {
            gameContainer.style.borderColor = pColor;
            gameContainer.style.boxShadow = `0 0 20px ${pColor}`;
        }

        ctx.clearRect(0, 0, canvas.width, canvas.height);

        if (time - lastSpawn > 1100) {
            obstacles.push({ x: canvas.width, w: 30, h: 40 });
            if(Math.random() > 0.3) {
                for(let i=0; i<3; i++) gameCoins.push({ x: canvas.width + 250 + (i*45), y: 320, collected: false });
            }
            lastSpawn = time;
        }

        player.dy += player.gravity; player.y += player.dy;
        if (player.y >= 315) { player.y = 315; player.dy = 0; player.jumps = 0; player.rot = 0; } else { player.rot += 0.15; }

        ctx.save(); ctx.translate(player.x + 17.5, player.y + 17.5); ctx.rotate(player.rot); ctx.fillStyle = pColor;
        if(graphicsLvl === 'pro') { ctx.shadowBlur = 15; ctx.shadowColor = pColor; }
        ctx.fillRect(-17.5, -17.5, 35, 35); ctx.restore();

        ctx.strokeStyle = pColor; ctx.lineWidth = 2; ctx.strokeRect(0, 350, canvas.width, 1);

        particles.forEach((p, i) => { p.x += p.vx; p.y += p.vy; p.life -= 0.05; if(p.life <= 0) particles.splice(i,1); ctx.fillStyle = p.color; ctx.globalAlpha = p.life; ctx.fillRect(p.x, p.y, 4, 4); });
        ctx.globalAlpha = 1;

        gameCoins.forEach(m => {
            m.x -= 6.5;
            if(!m.collected) {
                ctx.fillStyle = "#ffd700"; ctx.beginPath(); ctx.arc(m.x, m.y, 8, 0, Math.PI*2); ctx.fill();
                if(player.x < m.x + 15 && player.x + 35 > m.x - 15 && player.y < m.y + 15 && player.y + 35 > m.y - 15) {
                    m.collected = true; coins++; sync(); playCoinSound(); createCollectEffect(m.x, m.y);
                }
            }
        });

        obstacles.forEach((o, i) => {
            o.x -= 7;
            ctx.fillStyle = "#ff0044"; ctx.beginPath(); ctx.moveTo(o.x, 350); ctx.lineTo(o.x+15, 310); ctx.lineTo(o.x+30, 350); ctx.fill();
            if(player.x+25 > o.x && player.x < o.x+25 && player.y+25 > 310) { gameActive = false; fecharMenus(); }
            if(o.x < -50) { obstacles.splice(i, 1); score++; document.getElementById('score').innerText = score; }
        });
    }

    const jump = (e) => { if(e) e.preventDefault(); if(gameActive && (player.y >= 310 || player.jumps < 1)) { player.dy = player.jumpForce; player.jumps++; } };
    window.addEventListener('keydown', (e) => { if(e.code === 'Space') jump(); });
    canvas.addEventListener('touchstart', jump);
    sync();
</script>
</body>
</html>
