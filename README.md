
<!DOCTYPE html>
<html lang="id">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Holey Quest</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <style>
        /* Gaya khusus untuk papan permainan */
        #game-board.playing {
            cursor: none; /* Menyembunyikan kursor asli saat bermain */
        }
        
        .hole {
            position: absolute;
            background: radial-gradient(circle at 30% 30%, #444, #000 70%);
            border-radius: 50%;
            box-shadow: inset 0px 5px 15px rgba(0,0,0,0.9), 0px 2px 2px rgba(255,255,255,0.2);
        }

        #fake-cursor {
            position: absolute;
            width: 16px;
            height: 16px;
            background-color: #ff0000;
            border-radius: 50%;
            pointer-events: none; /* Agar tidak memblokir event mouse */
            z-index: 50;
            box-shadow: 0 0 10px #ff0000, 0 0 20px #ff0000;
            transform: translate(-50%, -50%); /* Center kursor di koordinat X,Y */
            display: none; /* Sembunyikan sampai game dimulai */
        }

        .zone {
            position: absolute;
            width: 80px;
            height: 80px;
            display: flex;
            align-items: center;
            justify-content: center;
            font-weight: bold;
            font-size: 1rem;
            color: white;
            text-shadow: 1px 1px 2px rgba(0,0,0,0.5);
            z-index: 10;
        }

        #start-zone {
            bottom: 0;
            left: 0;
            background: linear-gradient(135deg, #10b981, #047857);
            border-top-right-radius: 20px;
            box-shadow: 2px -2px 10px rgba(0,0,0,0.2);
            cursor: pointer;
        }

        #end-zone {
            top: 0;
            right: 0;
            background: linear-gradient(135deg, #fbbf24, #d97706);
            border-bottom-left-radius: 20px;
            box-shadow: -2px 2px 10px rgba(0,0,0,0.2);
        }
    </style>
</head>
<body class="bg-gray-900 text-white min-h-screen flex flex-col items-center justify-center overflow-hidden font-sans">

    <!-- Header & Status -->
    <div class="mb-4 text-center">
        <h1 class="text-4xl font-extrabold text-transparent bg-clip-text bg-gradient-to-r from-blue-400 to-emerald-400 mb-2">Holey Quest</h1>
        <p id="status-text" class="text-gray-300 text-lg">Klik "Mulai" lalu arahkan kursor ke area START.</p>
    </div>

    <!-- Papan Permainan -->
    <div id="game-container" class="relative w-[95vw] max-w-[800px] h-[65vh] max-h-[600px] bg-gray-300 rounded-xl border-4 border-gray-600 shadow-2xl overflow-hidden touch-none">
        
        <div id="game-board" class="w-full h-full relative">
            <div id="start-zone" class="zone">START</div>
            <div id="end-zone" class="zone">END</div>
            
            <!-- Tempat lubang-lubang akan di-generate -->
            <div id="holes-container"></div>

            <!-- Kursor palsu yang gemeteran -->
            <div id="fake-cursor"></div>
        </div>

        <!-- Overlay Menu (Mulai / Game Over / Win) -->
        <div id="overlay" class="absolute inset-0 bg-black/80 flex flex-col items-center justify-center z-50 backdrop-blur-sm transition-opacity duration-300">
            <h2 id="overlay-title" class="text-5xl font-bold mb-4 text-white">Holey Quest</h2>
            <p id="overlay-desc" class="text-gray-300 mb-8 max-w-md text-center">Arahkan kursor dari START ke END. Awas! Kursor kamu sedang sakit tremor dan di jalan banyak lubang. Jangan sampai jatuh!</p>
            <button id="btn-action" class="px-8 py-3 bg-blue-600 hover:bg-blue-500 text-white font-bold rounded-full text-xl shadow-[0_0_15px_rgba(37,99,235,0.5)] transition-transform hover:scale-105 active:scale-95">
                Mulai Main
            </button>
        </div>
    </div>

    <script>
        const board = document.getElementById('game-board');
        const holesContainer = document.getElementById('holes-container');
        const fakeCursor = document.getElementById('fake-cursor');
        const overlay = document.getElementById('overlay');
        const overlayTitle = document.getElementById('overlay-title');
        const overlayDesc = document.getElementById('overlay-desc');
        const btnAction = document.getElementById('btn-action');
        const statusText = document.getElementById('status-text');
        const startZone = document.getElementById('start-zone');

        // State Permainan
        let gameState = 'IDLE'; // IDLE, READY, PLAYING, GAMEOVER, WIN
        let realX = 0;
        let realY = 0;
        let fakeX = 0;
        let fakeY = 0;
        let lastRealX = 0;
        let lastRealY = 0;
        let currentTremble = 0;
        const fakeCursorRadius = 8;
        
        // Pengaturan Kesulitan
        const trembleIntensity = 8; // Diturunkan dari 18 agar lebih mudah
        const holeCount = 45; // Jumlah lubang
        let holesData = [];

        // Dimensi papan (Dibuat dinamis agar responsif)
        let boardWidth = 800;
        let boardHeight = 600;
        const zoneSize = 80;

        // Inisialisasi Event Listeners
        btnAction.addEventListener('click', prepareGame);
        
        function updateCoordinates(clientX, clientY) {
            const rect = board.getBoundingClientRect();
            realX = clientX - rect.left;
            realY = clientY - rect.top;

            // Logika untuk memulai jika state READY dan masuk start zone
            if (gameState === 'READY') {
                if (realX >= 0 && realX <= zoneSize && realY >= boardHeight - zoneSize && realY <= boardHeight) {
                    startGame();
                }
            }
        }

        // Dukungan untuk Mouse (Desktop)
        board.addEventListener('mousemove', (e) => {
            updateCoordinates(e.clientX, e.clientY);
        });

        // Dukungan untuk Touch (HP)
        board.addEventListener('touchmove', (e) => {
            e.preventDefault(); // Mencegah scrolling layar HP saat main
            updateCoordinates(e.touches[0].clientX, e.touches[0].clientY);
        }, { passive: false });

        board.addEventListener('touchstart', (e) => {
            if (gameState === 'READY' || gameState === 'PLAYING') {
                e.preventDefault();
                updateCoordinates(e.touches[0].clientX, e.touches[0].clientY);
            }
        }, { passive: false });

        // Jika mouse keluar dari papan saat bermain -> Game Over
        board.addEventListener('mouseleave', () => {
            if (gameState === 'PLAYING') {
                gameOver("Kursor keluar dari arena permainan!");
            }
        });

        // Jika jari diangkat dari layar HP saat bermain -> Game Over
        board.addEventListener('touchend', () => {
            if (gameState === 'PLAYING') {
                gameOver("Jari kamu terlepas dari layar!");
            }
        });

        function generateHoles() {
            // Ambil ukuran layar aktual sebelum membuat lubang
            boardWidth = board.clientWidth;
            boardHeight = board.clientHeight;

            holesContainer.innerHTML = '';
            holesData = [];

            for (let i = 0; i < holeCount; i++) {
                // Radius lubang acak antara 15 hingga 45
                let r = Math.random() * 30 + 15;
                
                // Posisi acak, pastikan tidak keluar batas
                let x = Math.random() * (boardWidth - r * 2) + r;
                let y = Math.random() * (boardHeight - r * 2) + r;

                // Jangan taruh lubang di area Start (kiri bawah)
                if (x < zoneSize + 40 && y > boardHeight - (zoneSize + 40)) continue;
                // Jangan taruh lubang di area End (kanan atas)
                if (x > boardWidth - (zoneSize + 40) && y < zoneSize + 40) continue;

                holesData.push({ x, y, r });

                const holeEl = document.createElement('div');
                holeEl.className = 'hole';
                holeEl.style.width = (r * 2) + 'px';
                holeEl.style.height = (r * 2) + 'px';
                holeEl.style.left = (x - r) + 'px';
                holeEl.style.top = (y - r) + 'px';
                holesContainer.appendChild(holeEl);
            }
        }

        function prepareGame() {
            gameState = 'READY';
            overlay.classList.add('hidden');
            fakeCursor.style.display = 'none';
            board.classList.remove('playing');
            statusText.innerHTML = 'Arahkan mouse kamu ke kotak <span class="text-green-500 font-bold">START</span> untuk memulai!';
            generateHoles();
        }

        function startGame() {
            gameState = 'PLAYING';
            board.classList.add('playing'); // Menyembunyikan kursor asli
            fakeCursor.style.display = 'block';
            statusText.innerHTML = '<span class="text-red-500 font-bold animate-pulse">Hati-hati! Tremor aktif saat bergerak! Bawa ke kotak END!</span>';
            
            // Set posisi awal kursor palsu agar tidak loncat dari jauh
            fakeX = realX;
            fakeY = realY;
            lastRealX = realX;
            lastRealY = realY;
            currentTremble = 0;

            requestAnimationFrame(gameLoop);
        }

        function gameLoop() {
            if (gameState !== 'PLAYING') return;

            // --- LOGIKA KURSOR GEMETERAN ---
            // Cek apakah mouse sedang bergerak
            let isMoving = (realX !== lastRealX || realY !== lastRealY);
            
            // Jika bergerak, terapkan tremor. Jika diam, tremor diredam (berhenti perlahan).
            if (isMoving) {
                currentTremble = trembleIntensity;
            } else {
                currentTremble = currentTremble * 0.8; // Easing agar berhentinya mulus
                if (currentTremble < 0.5) currentTremble = 0;
            }

            // Acak offset X dan Y berdasarkan intensitas
            let offsetX = (Math.random() - 0.5) * 2 * currentTremble;
            let offsetY = (Math.random() - 0.5) * 2 * currentTremble;

            // Update posisi kursor palsu
            fakeX = realX + offsetX;
            fakeY = realY + offsetY;

            // Simpan posisi terakhir untuk frame berikutnya
            lastRealX = realX;
            lastRealY = realY;

            // Render ke layar
            fakeCursor.style.left = fakeX + 'px';
            fakeCursor.style.top = fakeY + 'px';

            // --- DETEKSI TABRAKAN ---
            checkCollisions();

            // Loop terus
            if (gameState === 'PLAYING') {
                requestAnimationFrame(gameLoop);
            }
        }

        function checkCollisions() {
            // 1. Cek tabrakan dengan lubang (Lingkaran vs Lingkaran)
            for (let hole of holesData) {
                // Rumus jarak antara dua titik (pusat kursor palsu dan pusat lubang)
                let dx = fakeX - hole.x;
                let dy = fakeY - hole.y;
                let distance = Math.sqrt(dx * dx + dy * dy);

                // Jika jarak < jumlah radius mereka, berarti tabrakan!
                if (distance < (fakeCursorRadius + hole.r - 2)) { // -2 sebagai toleransi tipis
                    gameOver("Ups! Kursor kamu jatuh ke lubang.");
                    return;
                }
            }

            // 2. Cek apakah mencapai area END (Kanan Atas)
            if (fakeX >= boardWidth - zoneSize && fakeY <= zoneSize) {
                gameWin();
            }
        }

        function gameOver(reason) {
            gameState = 'GAMEOVER';
            board.classList.remove('playing'); // Kembalikan kursor asli
            fakeCursor.style.display = 'none';

            overlayTitle.textContent = "GAME OVER";
            overlayTitle.className = "text-5xl font-bold mb-4 text-red-500";
            overlayDesc.textContent = reason;
            btnAction.textContent = "Coba Lagi";
            btnAction.className = "px-8 py-3 bg-red-600 hover:bg-red-500 text-white font-bold rounded-full text-xl shadow-[0_0_15px_rgba(220,38,38,0.5)] transition-transform hover:scale-105 active:scale-95";
            
            overlay.classList.remove('hidden');
            statusText.textContent = "Yah, gagal...";
        }

        function gameWin() {
            gameState = 'WIN';
            board.classList.remove('playing');
            fakeCursor.style.display = 'none';

            overlayTitle.textContent = "KAMU MENANG!";
            overlayTitle.className = "text-5xl font-bold mb-4 text-yellow-400";
            overlayDesc.textContent = "Luar biasa! Kamu berhasil menaklukkan jalur lubang meski kursornya tremor hebat!";
            btnAction.textContent = "Main Lagi";
            btnAction.className = "px-8 py-3 bg-green-600 hover:bg-green-500 text-white font-bold rounded-full text-xl shadow-[0_0_15px_rgba(22,163,74,0.5)] transition-transform hover:scale-105 active:scale-95";
            
            overlay.classList.remove('hidden');
            statusText.textContent = "Pemenang!";
        }
    </script>
</body>
</html>
