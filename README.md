<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Pulse Detector</title>
    <style>
        :root {
            --primary: #ff4444;
            --bg: #1a1a1a;
        }

        * {
            box-sizing: border-box;
            font-family: -apple-system, sans-serif;
        }

        body {
            margin: 0;
            padding: 10px;
            background: var(--bg);
            color: white;
            height: 100vh;
            overflow: hidden;
        }

        .container {
            display: grid;
            grid-template-rows: auto 1fr auto;
            height: 100%;
            gap: 10px;
        }

        #videoFeed {
            width: 100%;
            border-radius: 12px;
            transform: scaleX(-1);
            background: black;
        }

        .stats {
            background: #2a2a2a;
            padding: 15px;
            border-radius: 12px;
            font-size: 14px;
        }

        .log {
            background: #2a2a2a;
            padding: 15px;
            border-radius: 12px;
            overflow-y: auto;
            max-height: 150px;
            font-size: 12px;
        }

        .pulse-indicator {
            width: 20px;
            height: 20px;
            border-radius: 50%;
            background: var(--primary);
            animation: pulse 1s infinite;
        }

        @keyframes pulse {
            0% { transform: scale(0.9); opacity: 0.7; }
            50% { transform: scale(1.1); opacity: 1; }
            100% { transform: scale(0.9); opacity: 0.7; }
        }
    </style>
</head>
<body>
    <div class="container">
        <div class="header">
            <h1>Pulse Detector 🔴</h1>
            <div class="pulse-indicator" style="display: none;"></div>
        </div>
        
        <video id="videoFeed" playsinline autoplay></video>
        
        <div class="stats">
            <div>Current RGB: <span id="currentRGB">-</span></div>
            <div>Pulse Duration: <span id="duration">0.00s</span></div>
            <div>Last Message: <span id="lastMessage">-</span></div>
        </div>

        <div class="log" id="messageLog"></div>
    </div>

    <script>
        const MESSAGE_RULES = [
            {name: 'SEND HELP 🆘', color: [255,0,0], duration: [1.3,1.7]},
            {name: 'ALL CLEAR ✅', color: [0,255,0], duration: [0.4,0.6]},
            {name: 'BACKUP 🔵', color: [0,0,255], duration: [2.5,3.5]}
        ];

        let isPulsing = false;
        let pulseStart = 0;
        let initialColor = [0,0,0];
        let lastDetection = 0;

        async function initCamera() {
            const video = document.getElementById('videoFeed');
            const stream = await navigator.mediaDevices.getUserMedia({
                video: {
                    facingMode: 'environment',
                    width: { ideal: 640 },
                    height: { ideal: 480 }
                }
            });
            video.srcObject = stream;
            await new Promise(resolve => video.onloadedmetadata = resolve);
            video.play();
            processVideo();
        }

        function processVideo() {
            const video = document.getElementById('videoFeed');
            const canvas = document.createElement('canvas');
            const ctx = canvas.getContext('2d');
            
            canvas.width = video.videoWidth;
            canvas.height = video.videoHeight;

            function analyzeFrame() {
                ctx.drawImage(video, 0, 0);
                const imageData = ctx.getImageData(0, 0, canvas.width, canvas.height);
                const rgb = getDominantColor(imageData);
                
                updateDisplay(rgb);
                checkPulse(rgb);
                requestAnimationFrame(analyzeFrame);
            }

            analyzeFrame();
        }

        function getDominantColor(imageData) {
            const data = imageData.data;
            let r=0, g=0, b=0, count=0;

            for(let i=0; i<data.length; i+=4) {
                if(data[i+3] > 128) { // Ignore transparent pixels
                    r += data[i];
                    g += data[i+1];
                    b += data[i+2];
                    count++;
                }
            }

            return count > 0 ? [
                Math.round(r/count),
                Math.round(g/count),
                Math.round(b/count)
            ] : [0,0,0];
        }

        function checkPulse(currentRGB) {
            const now = Date.now()/1000;
            
            if(colorDistance(currentRGB, [255,255,255]) < 50) return;

            if(colorDistance(currentRGB, initialColor) > 50 && isPulsing) {
                endPulse(currentRGB, true);
                return;
            }

            if(!isPulsing && colorDistance(currentRGB, [0,0,0]) > 100) {
                startPulse(currentRGB);
            } else if(isPulsing && now - lastDetection > 0.1) {
                endPulse(currentRGB);
            }
        }

        function startPulse(color) {
            isPulsing = true;
            pulseStart = Date.now()/1000;
            initialColor = [...color];
            document.querySelector('.pulse-indicator').style.display = 'block';
        }

        function endPulse(color, aborted=false) {
            isPulsing = false;
            const duration = (Date.now()/1000 - pulseStart).toFixed(2);
            const message = classifyPulse(initialColor, duration, aborted);
            
            logMessage(message, duration, initialColor);
            document.querySelector('.pulse-indicator').style.display = 'none';
            lastDetection = Date.now()/1000;
        }

        function classifyPulse(color, duration, aborted) {
            const match = MESSAGE_RULES.find(rule => 
                colorDistance(color, rule.color) < 50 &&
                duration >= rule.duration[0] &&
                duration <= rule.duration[1]
            );
            
            return match ? 
                `${match.name}${aborted ? ' (ABORTED)' : ''}` : 
                'UNKNOWN SIGNAL';
        }

        function colorDistance(c1, c2) {
            return Math.sqrt(
                (c1[0]-c2[0])**2 +
                (c1[1]-c2[1])**2 +
                (c1[2]-c2[2])**2
            );
        }

        function updateDisplay(rgb) {
            document.getElementById('currentRGB').textContent = rgb.join(', ');
            document.getElementById('duration').textContent = 
                isPulsing ? `${(Date.now()/1000 - pulseStart).toFixed(2)}s` : '0.00s';
        }

        function logMessage(message, duration, color) {
            const log = document.getElementById('messageLog');
            const entry = document.createElement('div');
            entry.innerHTML = `
                <div style="color: rgb(${color.join(',')}); margin: 5px 0;">
                    ${message} - ${duration}s
                </div>
            `;
            log.prepend(entry);
            document.getElementById('lastMessage').textContent = message;
        }

        // Initialize when safe
        if(navigator.mediaDevices?.getUserMedia) {
            initCamera();
        } else {
            alert('Camera access not supported in this browser');
        }
    </script>
</body>
</html>
