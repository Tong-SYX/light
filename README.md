<!DOCTYPE html>
<html lang="zh-TW">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>CPLD EPM1270T144 十字路口紅綠燈模擬器</title>
    <style>
        body {
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            background-color: #1e1e24;
            color: #fff;
            margin: 0;
            padding: 20px;
            display: flex;
            flex-direction: column;
            align-items: center;
        }

        h1 {
            color: #4af;
            margin-bottom: 5px;
        }

        .subtitle {
            color: #aaa;
            font-size: 14px;
            margin-bottom: 20px;
        }

        .main-container {
            display: flex;
            gap: 40px;
            background: #2a2a35;
            padding: 30px;
            border-radius: 15px;
            box-shadow: 0 10px 30px rgba(0,0,0,0.5);
            flex-wrap: wrap;
            justify-content: center;
        }

        /* 儀表板控制區 */
        .control-panel {
            width: 300px;
            display: flex;
            flex-direction: column;
            gap: 15px;
        }

        .panel-box {
            background: #15151c;
            padding: 15px;
            border-radius: 8px;
            border-left: 4px solid #4af;
        }

        .panel-title {
            font-weight: bold;
            color: #4af;
            margin-bottom: 10px;
            font-size: 14px;
            text-transform: uppercase;
        }

        .status-value {
            font-size: 20px;
            font-family: monospace;
            color: #2ecc71;
        }

        .btn {
            background: #e74c3c;
            color: white;
            border: none;
            padding: 12px;
            border-radius: 5px;
            cursor: pointer;
            font-weight: bold;
            font-size: 16px;
            transition: background 0.2s;
            width: 100%;
        }
        .btn:active {
            background: #c0392b;
        }

        .slider-container input {
            width: 100%;
            margin-top: 8px;
        }

        /* 十字路口視覺化 */
        .intersection {
            width: 360px;
            height: 360px;
            background: #34495e;
            position: relative;
            border-radius: 10px;
            overflow: hidden;
            border: 4px solid #2c3e50;
        }

        /* 馬路 */
        .road-v {
            width: 100px;
            height: 100%;
            background: #2c3e50;
            position: absolute;
            left: 130px;
        }
        .road-h {
            width: 100%;
            height: 100px;
            background: #2c3e50;
            position: absolute;
            top: 130px;
        }

        /* 燈號群組 */
        .traffic-light-block {
            background: #111;
            padding: 5px;
            border-radius: 5px;
            position: absolute;
            display: flex;
            gap: 5px;
            border: 1px solid #555;
        }

        /* 個別燈號外觀 */
        .light {
            width: 18px;
            height: 18px;
            border-radius: 50%;
            background-color: #333;
            box-shadow: inset 0 0 5px rgba(0,0,0,0.8);
            transition: all 0.1s;
        }

        /* 燈號亮起顏色與發光特效 */
        .light.red.active { background-color: #ff3838; box-shadow: 0 0 15px #ff3838, inset 0 0 2px white; }
        .light.yellow.active { background-color: #ffb300; box-shadow: 0 0 15px #ffb300, inset 0 0 2px white; }
        .light.green.active { background-color: #2ecc71; box-shadow: 0 0 15px #2ecc71, inset 0 0 2px white; }

        /* 各燈號依據 CPLD 板子方向定位 */
        .light-N { top: 75px; left: 145px; flex-direction: row; } /* 北向 TR1 TY1 TG1 */
        .light-E { top: 170px; right: 75px; flex-direction: column; } /* 東向 TR2 TY2 TG2 */
        .light-S { bottom: 75px; left: 145px; flex-direction: row-reverse; } /* 南向 TG3 TY3 TR3 */
        .light-W { top: 170px; left: 75px; flex-direction: column-reverse; } /* 西向 TG4 TY4 TR4 */

        .direction-label {
            position: absolute;
            font-weight: bold;
            font-size: 18px;
            color: rgba(255,255,255,0.3);
        }
    </style>
</head>
<body>

    <h1>模擬紅綠燈 (Verilog FSM Simulation)</h1>
    <div class="subtitle">硬體目標: EPM1270T144c5 | 開發環境: Quartus II 13.0 SP1</div>

    <div class="main-container">
        
        <div class="control-panel">
            <div class="panel-box">
                <div class="panel-title">FSM State (目前狀態)</div>
                <div class="status-value" id="lbl-state">2'b00 (S0: 南北通)</div>
            </div>
            
            <div class="panel-box">
                <div class="panel-title">clk_cnt (分頻計數器)</div>
                <div class="status-value" id="lbl-clk-cnt">0 / 999</div>
            </div>

            <div class="panel-box">
                <div class="panel-title">sec_cnt (內部秒數計存器)</div>
                <div class="status-value" id="lbl-sec-cnt">1 秒</div>
            </div>

            <div class="panel-box slider-container">
                <div class="panel-title">外部輸入時脈 (CLK): <span id="lbl-hz" style="color:#2ecc71">1000</span> Hz</div>
                <div class="subtitle" style="margin:0">調整此滑桿體驗你問的「調高Hz變快」效果</div>
                <input type="range" id="hz-slider" min="1000" max="5000" step="100" value="1000">
            </div>

            <button class="btn" id="btn-reset">RESET 按鈕 (Pin 8 接地 0)</button>
        </div>

        <div class="intersection">
            <div class="road-v"></div>
            <div class="road-h"></div>
            
            <div class="direction-label" style="top:20px; left:172px;">N</div>
            <div class="direction-label" style="top:168px; right:20px;">E</div>
            <div class="direction-label" style="bottom:20px; left:173px;">S</div>
            <div class="direction-label" style="top:168px; left:20px;">W</div>

            <div class="traffic-light-block light-N">
                <div class="light red" id="TR1"></div>
                <div class="light yellow" id="TY1"></div>
                <div class="light green" id="TG1"></div>
            </div>

            <div class="traffic-light-block light-E">
                <div class="light red" id="TR2"></div>
                <div class="light yellow" id="TY2"></div>
                <div class="light green" id="TG2"></div>
            </div>

            <div class="traffic-light-block light-S">
                <div class="light green" id="TG3"></div>
                <div class="light yellow" id="TY3"></div>
                <div class="light red" id="TR3"></div>
            </div>

            <div class="traffic-light-block light-W">
                <div class="light green" id="TG4"></div>
                <div class="light yellow" id="TY4"></div>
                <div class="light red" id="TR4"></div>
            </div>
        </div>

    </div>

    <script>
        // 模擬 Verilog 中的暫存器
        let current_state = 0; // 2'b00
        let clk_cnt = 0;       // reg [9:0]
        let sec_cnt = 1;       // reg [3:0]
        
        let system_hz = 1000;  // 預設 1000 Hz
        let simInterval = null;

        // 狀態常數定義
        const S0_NS_GREEN = 0;
        const S1_NS_YELLOW = 1;
        const S2_EW_GREEN = 2;
        const S3_EW_YELLOW = 3;

        // DOM 元素
        const lblState = document.getElementById('lbl-state');
        const lblClkCnt = document.getElementById('lbl-clk-cnt');
        const lblSecCnt = document.getElementById('lbl-sec-cnt');
        const lblHz = document.getElementById('lbl-hz');
        const hzSlider = document.getElementById('hz-slider');
        const btnReset = document.getElementById('btn-reset');

        // 初始化模擬時脈核心
        function startSimulation() {
            if (simInterval) clearInterval(simInterval);
            
            // 由於網頁瀏覽器無法精準跑出真實 1000Hz 甚至 5000Hz 的每秒呼叫次數
            // 這裡採用等比例換算技術，用穩定的 20ms (每秒50次) 網頁計時器來精準模擬硬體行為
            const stepsPerTick = system_hz / 50; 

            simInterval = setInterval(() => {
                for(let i=0; i<stepsPerTick; i++) {
                    hardwareClockEdge();
                }
                updateUI();
            }, 20);
        }

        // 核心邏輯：重現 Verilog 程式碼中的 always @(posedge clk)
        function hardwareClockEdge() {
            // 1. 一秒脈衝產生器 (clk_cnt 累加)
            let one_sec_tick = false;
            if (clk_cnt >= 999) {
                clk_cnt = 0;
                one_sec_tick = true;
            } else {
                clk_cnt++;
            }

            // 2. 狀態機與計時器跳轉
            if (one_sec_tick) {
                switch(current_state) {
                    case S0_NS_GREEN:
                        if (sec_cnt >= 8) { current_state = S1_NS_YELLOW; sec_cnt = 1; }
                        else sec_cnt++;
                        break;
                    case S1_NS_YELLOW:
                        if (sec_cnt >= 4) { current_state = S2_EW_GREEN; sec_cnt = 1; }
                        else sec_cnt++;
                        break;
                    case S2_EW_GREEN:
                        if (sec_cnt >= 8) { current_state = S3_EW_YELLOW; sec_cnt = 1; }
                        else sec_cnt++;
                        break;
                    case S3_EW_YELLOW:
                        if (sec_cnt >= 4) { current_state = S0_NS_GREEN; sec_cnt = 1; }
                        else sec_cnt++;
                        break;
                }
            }
        }

        // 組合電路輸出解碼：重現 always @(*) 將狀態映射到實體 LED
        function updateUI() {
            // 更新數據儀表板
            let stateText = "";
            switch(current_state) {
                case 0: stateText = "2'b00 (S0: 南北綠，東西紅)"; break;
                case 1: stateText = "2'b01 (S1: 南北黃，東西紅)"; break;
                case 2: stateText = "2'b10 (S2: 南北紅，東西綠)"; break;
                case 3: stateText = "2'b11 (S3: 南北紅，東西黃)"; break;
            }
            lblState.innerText = stateText;
            lblClkCnt.innerText = `${clk_cnt} / 999`;
            lblSecCnt.innerText = `${sec_cnt} 秒`;

            // 清除所有燈號
            document.querySelectorAll('.light').forEach(l => l.classList.remove('active'));

            // 依據目前狀態點亮對應的高電位 LED
            if (current_state === S0_NS_GREEN) {
                document.getElementById('TG1').classList.add('active'); // 北綠
                document.getElementById('TG3').classList.add('active'); // 南綠
                document.getElementById('TR2').classList.add('active'); // 東紅
                document.getElementById('TR4').classList.add('active'); // 西紅
            } else if (current_state === S1_NS_YELLOW) {
                document.getElementById('TY1').classList.add('active'); // 北黃
                document.getElementById('TY3').classList.add('active'); // 南黃
                document.getElementById('TR2').classList.add('active'); // 東紅
                document.getElementById('TR4').classList.add('active'); // 西紅
            } else if (current_state === S2_EW_GREEN) {
                document.getElementById('TR1').classList.add('active'); // 北紅
                document.getElementById('TR3').classList.add('active'); // 南紅
                document.getElementById('TG2').classList.add('active'); // 東綠
                document.getElementById('TG4').classList.add('active'); // 西綠
            } else if (current_state === S3_EW_YELLOW) {
                document.getElementById('TR1').classList.add('active'); // 北紅
                document.getElementById('TR3').classList.add('active'); // 南紅
                document.getElementById('TY2').classList.add('active'); // 東黃
                document.getElementById('TY4').classList.add('active'); // 西黃
            }
        }

        // 模擬低電位 Reset 按鈕 (按下去為 0)
        btnReset.addEventListener('mousedown', () => {
            current_state = S0_NS_GREEN;
            clk_cnt = 0;
            sec_cnt = 1;
            updateUI();
        });

        // 調整外部赫茲頻率監聽
        hzSlider.addEventListener('input', (e) => {
            system_hz = parseInt(e.target.value);
            lblHz.innerText = system_hz;
            startSimulation();
        });

        // 啟動
        startSimulation();
    </script>
</body>
</html>
