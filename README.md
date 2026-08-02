<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>ТАНКОВЫЙ СТРИМ-HUD v4.3 // ВЕРСИЯ С КРАСИВЫМИ ЭФФЕКТАМИ</title>
    <!-- Подключаем библиотеку Centrifuge для WebSocket DonationAlerts -->
    <script src="https://cdn.jsdelivr.net/npm/centrifuge@2.8.4/dist/centrifuge.min.js"></script>
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Share+Tech+Mono&display=swap');

        :root {
            --hud-orange: #ff9f43; 
            --hud-cyan: #00cec9;   
            --hud-red: #ff3e3e;    
            --hud-green: #00ff7f;  
            --bg-opacity: 0.95; 
        }

        * {
            box-sizing: border-box;
            user-select: none;
        }

        body {
            margin: 0;
            padding: 0;
            background: transparent !important; 
            color: var(--hud-orange);
            font-family: 'Share Tech Mono', monospace;
            overflow-x: hidden;
            min-height: 100vh;
        }

        .top-nav {
            display: flex;
            justify-content: space-between;
            align-items: center;
            padding: 12px 30px;
            background: linear-gradient(180deg, #111 0%, rgba(0,0,0,0.85) 100%);
            border-bottom: 3px solid var(--hud-orange);
            box-shadow: 0 5px 20px rgba(255, 159, 67, 0.4);
            position: sticky; top: 0; z-index: 100;
        }

        .system-status {
            font-size: 18px; 
            letter-spacing: 2px;
            text-transform: uppercase;
            display: flex;
            align-items: center;
            gap: 10px;
        }

        .rec-dot {
            width: 10px;
            height: 10px;
            background-color: var(--hud-red);
            border-radius: 50%;
            box-shadow: 0 0 10px var(--hud-red);
            animation: blink 1s infinite;
        }

        .nav-tabs button {
            background: transparent;
            border: 2px solid var(--hud-orange);
            color: var(--hud-orange);
            padding: 10px 22px;
            font-family: 'Share Tech Mono', monospace;
            font-size: 18px; 
            cursor: pointer;
            margin-left: 10px;
            text-transform: uppercase;
            transition: all 0.3s;
            background: linear-gradient(135deg, rgba(255,159,67,0.1) 0%, rgba(0,0,0,0) 100%);
        }

        .nav-tabs button.active {
            background: var(--hud-orange);
            color: #000;
            box-shadow: 0 0 15px var(--hud-orange), inset 0 0 10px rgba(255,255,255,0.5);
            font-weight: bold;
        }

        .view-container {
            padding: 30px;
            max-width: 1400px;
            margin: 0 auto;
        }

        .view {
            display: none;
        }

        .view.active {
            display: block;
        }

        .hud-vertical-stack {
            display: flex;
            flex-direction: column;
            align-items: center;
            gap: 25px;
            margin-top: 15px;
        }

        /* КРАСИВАЯ ПАНЕЛЬ С СЕТКОЙ И ЭФФЕКТАМИ */
        .hud-panel {
            width: 100%;
            max-width: 600px;
            background: linear-gradient(rgba(0, 206, 201, 0.02) 1px, transparent 1px),
                        linear-gradient(90deg, rgba(0, 206, 201, 0.02) 1px, transparent 1px),
                        rgba(15, 20, 25, var(--bg-opacity)); 
            background-size: 15px 15px;
            border: 2px solid rgba(255, 159, 67, 0.4);
            border-radius: 8px;
            padding: 22px;
            box-shadow: 0 10px 35px rgba(0,0,0,0.6), inset 0 0 15px rgba(255, 159, 67, 0.1);
            position: relative;
            overflow: hidden;
        }

        /* Световой луч сканера на панелях */
        .hud-panel::after {
            content: '';
            position: absolute;
            left: 0; top: -100%;
            width: 100%; height: 20%;
            background: linear-gradient(to bottom, transparent, rgba(255, 159, 67, 0.05), transparent);
            animation: panelScan 6s infinite linear;
            pointer-events: none;
        }

        /* Акцентная полоса слева */
        .panel-side-line {
            position: absolute;
            left: 0; top: 15%;
            height: 70%; width: 3px;
            background: var(--hud-cyan);
            box-shadow: 0 0 10px var(--hud-cyan);
        }

        .hud-center {
            width: 100%;
            max-width: 600px;
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            position: relative;
            border: 4px solid var(--hud-orange);
            border-radius: 12px;
            padding: 35px 40px;
            background: radial-gradient(circle at center, rgba(15, 20, 25, var(--bg-opacity)) 0%, rgba(5, 5, 5, var(--bg-opacity)) 100%); 
            box-shadow: 0 0 50px rgba(255, 159, 67, 0.25), inset 0 0 40px rgba(255, 159, 67, 0.15);
        }

        .hud-center::before, .hud-center::after {
            content: '';
            position: absolute;
            width: 40px;
            height: 40px;
            border: 4px solid var(--hud-cyan);
            box-shadow: 0 0 15px var(--hud-cyan);
        }
        .hud-center::before { top: -4px; left: -4px; border-right: 0; border-bottom: 0; }
        .hud-center::after { bottom: -4px; right: -4px; border-left: 0; border-top: 0; }

        .target-status-label {
            font-size: 20px; 
            color: var(--hud-cyan);
            letter-spacing: 5px;
            margin-bottom: 15px;
            text-transform: uppercase;
            text-shadow: 0 0 10px var(--hud-cyan);
        }

        /* ТАКТИЧЕСКИЙ ПРИЦЕЛ */
        .tank-reticle-container {
            width: 220px;
            height: 220px;
            margin: 15px 0;
            position: relative;
            display: flex;
            align-items: center;
            justify-content: center;
            transition: all 0.4s ease;
        }

        .reticle-outer-ring {
            position: absolute;
            width: 190px;
            height: 190px;
            border: 2px dashed var(--hud-cyan);
            border-radius: 50%;
            box-shadow: 0 0 15px rgba(0, 206, 201, 0.2);
            animation: rotateClockwise 20s linear infinite;
        }

        .reticle-inner-circle {
            position: absolute;
            width: 110px;
            height: 110px;
            border: 1px double rgba(0, 206, 201, 0.4);
            border-radius: 50%;
            animation: pulseCircle 2s infinite ease-in-out;
        }

        .reticle-axis-h {
            position: absolute;
            width: 160px;
            height: 2px;
            background: linear-gradient(90deg, transparent, var(--hud-cyan) 20%, var(--hud-cyan) 80%, transparent);
        }
        .reticle-axis-v {
            position: absolute;
            width: 2px;
            height: 160px;
            background: linear-gradient(180deg, transparent, var(--hud-cyan) 20%, var(--hud-cyan) 80%, transparent);
        }

        .reticle-dot {
            position: absolute;
            width: 8px;
            height: 8px;
            background: var(--hud-cyan);
            border-radius: 50%;
            box-shadow: 0 0 10px var(--hud-cyan);
        }

        .reticle-bracket {
            position: absolute;
            width: 25px;
            height: 25px;
            border: 3px solid var(--hud-orange);
            transition: all 0.4s cubic-bezier(0.175, 0.885, 0.32, 1.275);
        }
        .b-tl { top: 0; left: 0; border-right: 0; border-bottom: 0; }
        .b-tr { top: 0; right: 0; border-left: 0; border-bottom: 0; }
        .b-bl { bottom: 0; left: 0; border-right: 0; border-top: 0; }
        .b-br { bottom: 0; right: 0; border-left: 0; border-top: 0; }

        .reticle-scanner-line {
            position: absolute;
            width: 170px;
            height: 2px;
            background: rgba(0, 206, 201, 0.6);
            box-shadow: 0 0 12px var(--hud-cyan);
            animation: scanMove 3s ease-in-out infinite;
        }

        /* Состояния прицела */
        .tank-reticle-container.locked { transform: scale(1.05); }
        .tank-reticle-container.locked .reticle-outer-ring { border-color: var(--hud-red); border-style: solid; width: 160px; height: 160px; box-shadow: 0 0 25px rgba(255, 62, 62, 0.5); animation-duration: 5s; }
        .tank-reticle-container.locked .reticle-axis-h, .tank-reticle-container.locked .reticle-axis-v { background: var(--hud-red); box-shadow: 0 0 8px var(--hud-red); }
        .tank-reticle-container.locked .reticle-dot { background: var(--hud-red); box-shadow: 0 0 15px var(--hud-red); transform: scale(1.4); }
        .tank-reticle-container.locked .reticle-bracket { border-color: var(--hud-red); width: 35px; height: 35px; }
        .tank-reticle-container.locked .reticle-scanner-line { background: rgba(255, 62, 62, 0.8); box-shadow: 0 0 15px var(--hud-red); animation-duration: 1s; }

        .tank-reticle-container.searching .reticle-outer-ring { border-color: var(--hud-green); }
        .tank-reticle-container.searching .reticle-axis-h, .tank-reticle-container.searching .reticle-axis-v { background: var(--hud-green); }
        .tank-reticle-container.searching .reticle-dot { background: var(--hud-green); box-shadow: 0 0 10px var(--hud-green); }
        .tank-reticle-container.searching .reticle-bracket { border-color: var(--hud-green); }
        .tank-reticle-container.searching .reticle-scanner-line { background: rgba(0, 255, 127, 0.5); box-shadow: 0 0 10px var(--hud-green); }

        .target-name {
            font-size: 46px; 
            font-weight: bold;
            color: #fff;
            text-shadow: 0 0 25px var(--hud-orange), 0 0 5px #fff;
            text-align: center;
            margin: 15px 0;
            letter-spacing: 3px;
        }

        .fire-button {
            background: transparent;
            border: 3px solid var(--hud-red);
            color: var(--hud-red);
            font-family: 'Share Tech Mono', monospace;
            font-size: 26px; 
            padding: 14px 45px;
            cursor: pointer;
            text-transform: uppercase;
            letter-spacing: 4px;
            transition: all 0.2s;
            position: relative;
            border-radius: 8px;
            background: linear-gradient(180deg, rgba(255,62,62,0.1) 0%, rgba(0,0,0,0) 100%);
        }
        .fire-button:hover { background: rgba(255, 62, 62, 0.25); box-shadow: 0 0 30px var(--hud-red), inset 0 0 15px rgba(255,255,255,0.4); color: #fff; }

        .panel-header {
            border-bottom: 2px solid var(--hud-orange);
            padding-bottom: 8px;
            margin-bottom: 18px;
            font-size: 24px; 
            letter-spacing: 2px;
            color: var(--hud-cyan);
            text-shadow: 0 0 10px var(--hud-cyan);
        }

        .next-preview {
            font-size: 36px; 
            color: #fff;
            margin-top: 12px;
            border-left: 6px solid var(--hud-cyan);
            padding-left: 18px;
            text-shadow: 0 0 15px #fff;
            font-weight: bold;
            letter-spacing: 1px;
        }

        .progress-bar-container {
            width: 100%;
            background: #1a222d;
            height: 18px; 
            border: 2px solid var(--hud-orange);
            margin-top: 15px;
            border-radius: 8px;
            overflow: hidden;
            box-shadow: 0 0 10px rgba(255, 159, 67, 0.2);
        }
        .progress-fill {
            width: 0%;
            height: 100%;
            background: linear-gradient(90deg, var(--hud-orange), #ffca28);
            box-shadow: 0 0 15px var(--hud-orange);
            transition: width 0.4s;
        }

        /* Декоративный статус-шум внизу панелей */
        .panel-deco-footer {
            display: flex;
            justify-content: space-between;
            font-size: 13px;
            color: rgba(0, 206, 201, 0.4);
            margin-top: 15px;
            border-top: 1px dashed rgba(0, 206, 201, 0.15);
            padding-top: 6px;
            letter-spacing: 1px;
        }

        /* Админка */
        .slider-container { display: flex; align-items: center; gap: 15px; }
        .slider-container input[type="range"] { flex-grow: 1; cursor: pointer; accent-color: var(--hud-cyan); }
        .admin-grid { display: grid; grid-template-columns: 1fr 2fr; gap: 30px; }
        .form-group { margin-bottom: 20px; }
        .form-group label { display: block; margin-bottom: 7px; font-size: 18px; color: var(--hud-cyan); }
        .form-group input, .form-group textarea { width: 100%; background: #0d1117; border: 2px solid var(--hud-orange); color: #fff; padding: 12px; font-family: 'Share Tech Mono', monospace; font-size: 18px; border-radius: 4px; }
        .btn-admin { background: var(--hud-cyan); color: #000; border: none; padding: 14px 25px; font-family: 'Share Tech Mono', monospace; font-size: 20px; font-weight: bold; cursor: pointer; text-transform: uppercase; width: 100%; border-radius: 4px; background: linear-gradient(135deg, #fff 0%, var(--hud-cyan) 20%, var(--hud-cyan) 80%, #000 100%); }
        .btn-admin:hover { box-shadow: 0 0 20px var(--hud-cyan); }
        .tank-table { width: 100%; border-collapse: collapse; margin-top: 15px; }
        .tank-table th, .tank-table td { border: 2px solid rgba(255, 159, 67, 0.25); padding: 14px; text-align: left; font-size: 16px; }
        .tank-table th { background: rgba(255, 159, 67, 0.15); color: var(--hud-cyan); text-transform: uppercase; }
        .tank-table tr.destroyed { text-decoration: line-through; color: #555; background: rgba(0,0,0,0.2); }
        .row-actions button { background: transparent; border: 2px solid var(--hud-red); color: var(--hud-red); cursor: pointer; padding: 6px 12px; font-size: 15px; border-radius: 4px; }
        .row-actions button:hover { background: var(--hud-red); color: #000;}
        .stats-blocks { display: grid; grid-template-columns: repeat(3, 1fr); gap: 20px; margin-bottom: 30px; }
        .stat-card { background: rgba(20, 24, 30, 0.9); border: 2px dashed var(--hud-cyan); padding: 20px; text-align: center; border-radius: 8px; }
        .stat-card div:first-child { font-size: 16px; color: var(--hud-cyan); text-transform: uppercase; letter-spacing: 1px;}
        .stat-card div:last-child { font-size: 36px; font-weight: bold; color: #fff; text-shadow: 0 0 10px #fff;}

        /* Статус подключения */
        .status-badge { font-size: 14px; padding: 4px 8px; border-radius: 4px; display: inline-block; margin-top: 5px; }
        .status-badge.connected { background: rgba(0,255,127,0.2); color: var(--hud-green); border: 1px solid var(--hud-green); }
        .status-badge.disconnected { background: rgba(255,62,62,0.2); color: var(--hud-red); border: 1px solid var(--hud-red); }

        /* АНИМАЦИИ */
        @keyframes rotateClockwise { from { transform: rotate(0deg); } to { transform: rotate(360deg); } }
        @keyframes pulseCircle { 0%, 100% { transform: scale(1); opacity: 0.4; } 50% { transform: scale(1.1); opacity: 0.8; } }
        @keyframes scanMove { 0%, 100% { top: 10%; opacity: 0; } 50% { opacity: 1; top: 90%; } }
        @keyframes blink { 0%, 100% { opacity: 1; } 50% { opacity: 0.2; } }
        @keyframes panelScan { 0% { top: -100%; } 30% { top: 100%; } 100% { top: 100%; } }
    </style>
</head>
<body>

<div class="top-nav">
    <div class="system-status">
        <div class="rec-dot"></div>
        ● СИНХРОНИЗАЦИЯ С OBS АКТИВНА // СТАТУС: СТАТИСТИКА ИГРЫ
    </div>
    <div class="nav-tabs">
        <button id="tab-hud" class="active" onclick="switchTab('hud')">ТАКТИЧЕСКИЙ HUD</button>
        <button id="tab-admin" onclick="switchTab('admin')">ПУНКТ УПРАВЛЕНИЯ [АДМИН]</button>
    </div>
</div>

<div class="view-container">

    <div id="view-hud" class="view active">
        <div class="hud-vertical-stack">
            
            <div class="hud-panel">
                <div class="panel-side-line"></div>
                <div class="panel-header">// ПОКАЗАТЕЛИ МИССИИ</div>
                <div style="display: flex; justify-content: space-between; margin-bottom: 12px; font-size: 24px; font-weight: bold;">
                    <div>КОНТРАКТОВ В ОЧЕРЕДИ: <span id="hud-left-count" style="color:#fff">0</span></div>
                    <div>ЛИКВИДИРОВАНО: <span id="hud-destroyed-count" style="color:var(--hud-green);">0</span></div>
                </div>
                
                <div class="progress-bar-container">
                    <div id="hud-progress-fill" class="progress-fill"></div>
                </div>
                <div id="hud-progress-txt" style="font-size: 20px; text-align: right; margin-top: 8px; color: var(--hud-cyan); font-weight: bold;">0% ВЫПОЛНЕНО</div>
                
                <div class="panel-deco-footer">
                    <div>[ ENGINE STATUS: NOMINAL ]</div>
                    <div>[ CORE_TEMP: 42°C ]</div>
                </div>
            </div>

            <div class="hud-center">
                <div class="target-status-label" id="target-status">// СИСТЕМА ГОТОВА //</div>
                
                <div class="tank-reticle-container searching" id="tank-reticle">
                    <div class="reticle-bracket b-tl"></div>
                    <div class="reticle-bracket b-tr"></div>
                    <div class="reticle-bracket b-bl"></div>
                    <div class="reticle-bracket b-br"></div>
                    
                    <div class="reticle-outer-ring"></div>
                    <div class="reticle-inner-circle"></div>
                    <div class="reticle-axis-h"></div>
                    <div class="reticle-axis-v"></div>
                    <div class="reticle-scanner-line"></div>
                    <div class="reticle-dot"></div>
                </div>

                <div class="target-name" id="hud-current-target">НЕТ АКТИВНЫХ ЦЕЛЕЙ</div>
                
                <button class="fire-button" onclick="confirmDestruction()">ЦЕЛЬ УНИЧТОЖЕНА</button>
            </div>

            <div class="hud-panel">
                <div class="panel-side-line" style="background: var(--hud-orange); box-shadow: 0 0 10px var(--hud-orange);"></div>
                <div class="panel-header">// ТЕЛЕМЕТРИЯ</div>
                <div style="font-size: 22px; color: var(--hud-cyan); text-transform: uppercase; letter-spacing: 1px; font-weight: bold;">СЛЕДУЮЩАЯ ЦЕЛЬ В ОЧЕРЕДИ:</div>
                <div class="next-preview" id="hud-next-target">НЕТ</div>
                
                <div class="panel-deco-footer">
                    <div>[ FEED: LIVE_STREAM_CONNECTED ]</div>
                    <div>[ MATRIX_LOC: SEC_07_B ]</div>
                </div>
            </div>

        </div>
    </div>

    <div id="view-admin" class="view">
        <div class="stats-blocks">
            <div class="stat-card">
                <div>ВСЕГО ЗАКАЗОВ</div>
                <div id="stat-total">0</div>
            </div>
            <div class="stat-card">
                <div>УНИЧТОЖЕНО</div>
                <div id="stat-done" style="color:var(--hud-green)">0</div>
            </div>
            <div class="stat-card">
                <div>ЭФФЕКТИВНОСТЬ</div>
                <div id="stat-ratio" style="color:var(--hud-cyan)">0%</div>
            </div>
        </div>

        <div class="admin-grid">
            <div class="hud-panel" style="height: auto;">
                <div class="panel-header">// НАСТРОЙКИ ИНТЕРФЕЙСА</div>
                <div class="form-group">
                    <label>Прозрачность панелей: <span id="opacity-value">95%</span></label>
                    <div class="slider-container">
                        <input type="range" id="input-opacity" min="0" max="100" value="95" oninput="changeOpacity(this.value)">
                    </div>
                </div>

                <!-- БЛОК НАСТРОЙКИ DONATIONALERTS -->
                <div class="panel-header" style="margin-top: 25px;">// DONATIONALERTS ИНТЕГРАЦИЯ</div>
                <div class="form-group">
                    <label>ID пользователя (USER ID)</label>
                    <input type="text" id="input-da-userid" placeholder="Например: 1234567">
                </div>
                <div class="form-group">
                    <label>Токен виджетов (SOCKET TOKEN)</label>
                    <input type="password" id="input-da-token" placeholder="Токен из настроек DA">
                </div>
                <button class="btn-admin" onclick="saveDASettings()">Сохранить настройки DA</button>
                <div id="da-status-badge" class="status-badge disconnected">НЕ ПОДКЛЮЧЕНО</div>

                <div class="panel-header" style="margin-top: 25px;">// ДОБАВИТЬ ЗАКАЗ ВРУЧНУЮ</div>
                <div class="form-group">
                    <label>Название танка / Цели</label>
                    <input type="text" id="input-name" placeholder="Например: Объект 277">
                </div>
                <button class="btn-admin" onclick="addSingleTank()">Внести в реестр</button>

                <div class="panel-header" style="margin-top: 30px;">// МАССОВЫЙ ИМПОРТ ИЗ БЛОКНОТА</div>
                <div class="form-group">
                    <textarea id="input-bulk" rows="5" placeholder="Вставь список танков, каждый с новой строки..."></textarea>
                </div>
                <button class="btn-admin" style="background: var(--hud-orange); color: #000;" onclick="importBulk()">Добавить список пакетно</button>
                
                <button class="btn-admin" style="background: var(--hud-red); margin-top: 25px; color: #fff; box-shadow: 0 0 10px var(--hud-red);" onclick="clearAllData()">СБРОСИТЬ ВСЮ БАЗУ ДАННЫХ</button>
            </div>

            <div class="hud-panel" style="overflow-y: auto; max-height: 650px; height: auto;">
                <div class="panel-header">// ТЕКУЩИЙ РЕЕСТР И ОЧЕРЕДНОСТЬ ЗАКАЗОВ</div>
                <table class="tank-table">
                    <thead>
                        <tr>
                            <th>№</th>
                            <th>Название техники</th>
                            <th>Статус</th>
                            <th>Действия</th>
                        </tr>
                    </thead>
                    <tbody id="tank-list-rows"></tbody>
                </table>
            </div>
        </div>
    </div>

</div>

<script>
    let db = JSON.parse(localStorage.getItem('tank_hud_db_v4_3')) || [];
    let savedOpacity = localStorage.getItem('tank_hud_opacity') || '0.95';
    let centrifugeInstance = null;

    // ==========================================
    // ЛОГИКА DONATIONALERTS
    // ==========================================
    function saveDASettings() {
        const userId = document.getElementById('input-da-userid').value.trim();
        const token = document.getElementById('input-da-token').value.trim();

        localStorage.setItem('tank_hud_da_userid', userId);
        localStorage.setItem('tank_hud_da_token', token);

        playBeep('add');
        initDonationAlerts();
    }

    function initDonationAlerts() {
        const userId = localStorage.getItem('tank_hud_da_userid') || '';
        const token = localStorage.getItem('tank_hud_da_token') || '';
        const badge = document.getElementById('da-status-badge');

        // Заполняем инпуты сохраненными значениями
        document.getElementById('input-da-userid').value = userId;
        document.getElementById('input-da-token').value = token;

        // Если предыдущее соединение существует — отключаем его
        if (centrifugeInstance) {
            centrifugeInstance.disconnect();
            centrifugeInstance = null;
        }

        if (!userId || !token) {
            badge.innerText = "ОЖИДАЕТСЯ ВВОД ДАННЫХ DA";
            badge.className = "status-badge disconnected";
            return;
        }

        try {
            centrifugeInstance = new Centrifuge('wss://centrifugo.donationalerts.com/connection/websocket');
            centrifugeInstance.setToken(token);

            centrifugeInstance.subscribe(`$donations:${userId}`, function(message) {
                const donation = message.data;
                
                // Извлекаем название танка из текста доната
                if (donation && donation.message) {
                    addTankFromDonation(donation.message);
                }
            });

            centrifugeInstance.on('connect', function() {
                badge.innerText = "ПОДКЛЮЧЕНО К DONATIONALERTS";
                badge.className = "status-badge connected";
            });

            centrifugeInstance.on('disconnect', function() {
                badge.innerText = "ОШИБКА ПОДКЛЮЧЕНИЯ DA";
                badge.className = "status-badge disconnected";
            });

            centrifugeInstance.connect();
        } catch(e) {
            console.error("Ошибка при запуске Centrifuge:", e);
            badge.innerText = "ОШИБКА СКРИПТА DA";
            badge.className = "status-badge disconnected";
        }
    }

    function addTankFromDonation(tankName) {
        const cleanName = tankName.trim();
        if (!cleanName) return;

        db.push({
            id: Date.now() + Math.random(),
            name: cleanName,
            status: 'pending'
        });

        playBeep('add');
        saveToStorage();
    }
    // ==========================================

    function playBeep(type) {
        try {
            const ctx = new (window.AudioContext || window.webkitAudioContext)();
            const osc = ctx.createOscillator();
            const gain = ctx.createGain();
            osc.connect(gain);
            gain.connect(ctx.destination);

            if (type === 'kill') {
                osc.type = 'sawtooth';
                osc.frequency.setValueAtTime(120, ctx.currentTime);
                osc.frequency.exponentialRampToValueAtTime(500, ctx.currentTime + 0.35);
                gain.gain.setValueAtTime(0.25, ctx.currentTime);
                gain.gain.exponentialRampToValueAtTime(0.01, ctx.currentTime + 0.35);
                osc.start(); osc.stop(ctx.currentTime + 0.35);
            } else if (type === 'tab' || type === 'add') {
                osc.type = 'sine';
                osc.frequency.setValueAtTime(750, ctx.currentTime);
                gain.gain.setValueAtTime(0.1, ctx.currentTime);
                gain.gain.exponentialRampToValueAtTime(0.01, ctx.currentTime + 0.15);
                osc.start(); osc.stop(ctx.currentTime + 0.15);
            }
        } catch(e) { console.log("Audio Error"); }
    }

    function changeOpacity(val) {
        let alpha = val / 100;
        document.documentElement.style.setProperty('--bg-opacity', alpha);
        document.getElementById('opacity-value').innerText = val + '%';
        localStorage.setItem('tank_hud_opacity', alpha);
    }

    function initOpacity() {
        let percent = Math.round(savedOpacity * 100);
        document.documentElement.style.setProperty('--bg-opacity', savedOpacity);
        document.getElementById('input-opacity').value = percent;
        document.getElementById('opacity-value').innerText = percent + '%';
    }

    function saveToStorage() {
        localStorage.setItem('tank_hud_db_v4_3', JSON.stringify(db));
        renderUI();
    }

    function switchTab(target) {
        playBeep('tab');
        document.querySelectorAll('.view').forEach(v => v.classList.remove('active'));
        document.querySelectorAll('.nav-tabs button').forEach(b => b.classList.remove('active'));
        
        document.getElementById(`view-${target}`).classList.add('active');
        document.getElementById(`tab-${target}`).classList.add('active');
    }

    function addSingleTank() {
        const nameInput = document.getElementById('input-name');
        if (!nameInput.value.trim()) return;

        addTankFromDonation(nameInput.value);
        nameInput.value = '';
    }

    function importBulk() {
        const bulkText = document.getElementById('input-bulk').value;
        if (!bulkText.trim()) return;

        const lines = bulkText.split('\n').map(l => l.trim()).filter(l => l.length > 0);
        lines.forEach(line => {
            db.push({
                id: Date.now() + Math.random(),
                name: line,
                status: 'pending'
            });
        });

        document.getElementById('input-bulk').value = '';
        playBeep('add');
        saveToStorage();
    }

    function deleteTank(id) {
        db = db.filter(item => item.id !== id);
        saveToStorage();
    }

    function clearAllData() {
        if (confirm("Очистить всю базу танков?")) {
            db = [];
            saveToStorage();
        }
    }

    function confirmDestruction() {
        const currentTarget = db.find(item => item.status === 'pending');
        if (currentTarget) {
            currentTarget.status = 'destroyed';
            playBeep('kill');
            saveToStorage();
        }
    }

    window.addEventListener('keydown', function(e) {
        if (e.code === 'Space' && document.activeElement.tagName !== 'INPUT' && document.activeElement.tagName !== 'TEXTAREA') {
            e.preventDefault(); 
            confirmDestruction();
        }
    });

    function renderUI() {
        const activeTargets = db.filter(item => item.status === 'pending');
        const destroyedTargets = db.filter(item => item.status === 'destroyed');
        
        const currentField = document.getElementById('hud-current-target');
        const statusLabel = document.getElementById('target-status');
        const reticle = document.getElementById('tank-reticle');

        if (activeTargets.length > 0) {
            currentField.innerText = activeTargets[0].name.toUpperCase();
            statusLabel.innerText = "// ЦЕЛЬ ЗАХВАЧЕНА //";
            statusLabel.style.color = "var(--hud-red)";
            
            reticle.className = "tank-reticle-container locked";

            if (activeTargets.length > 1) {
                document.getElementById('hud-next-target').innerText = activeTargets[1].name.toUpperCase();
            } else {
                document.getElementById('hud-next-target').innerText = "ПОСЛЕДНИЙ ЗАКАЗ";
            }
        } else {
            currentField.innerText = "НЕТ АКТИВНЫХ ЦЕЛЕЙ";
            statusLabel.innerText = "// ПОИСК СИГНАЛА //";
            statusLabel.style.color = "var(--hud-green)";
            
            reticle.className = "tank-reticle-container searching";
            
            document.getElementById('hud-next-target').innerText = "НЕТ";
        }

        document.getElementById('hud-left-count').innerText = activeTargets.length;
        document.getElementById('hud-destroyed-count').innerText = destroyedTargets.length;

        const totalCount = db.length;
        const ratio = totalCount > 0 ? Math.round((destroyedTargets.length / totalCount) * 100) : 0;
        document.getElementById('hud-progress-txt').innerText = `${ratio}% ВЫПОЛНЕНО`;
        document.getElementById('hud-progress-fill').style.width = `${ratio}%`;

        document.getElementById('stat-total').innerText = totalCount;
        document.getElementById('stat-done').innerText = destroyedTargets.length;
        document.getElementById('stat-ratio').innerText = `${ratio}%`;

        const tbody = document.getElementById('tank-list-rows');
        tbody.innerHTML = '';

        db.forEach((index, item) => {})
        db.forEach((item, index) => {
            const tr = document.createElement('tr');
            if (item.status === 'destroyed') tr.className = 'destroyed';
            
            tr.innerHTML = `
                <td>${index + 1}</td>
                <td style="color:#fff; font-weight:bold">${item.name}</td>
                <td style="color: ${item.status === 'destroyed' ? 'var(--hud-green)' : 'var(--hud-orange)'}">
                    ${item.status === 'destroyed' ? 'УНИЧТОЖЕН' : 'В ОЧЕРЕДИ'}
                </td>
                <td class="row-actions">
                    <button onclick="deleteTank(${item.id})">Удалить</button>
                </td>
            `;
            tbody.appendChild(tr);
        });
    }

    initOpacity();
    renderUI();
    initDonationAlerts(); // Автоматически загружает сохраненный токен при открытии
</script>

</body>
</html>
