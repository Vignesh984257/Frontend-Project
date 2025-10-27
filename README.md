// server.js (Node.js)

const WebSocket = require('ws');
const http = require('http');

// A simple map to hold mock stock prices
const stocks = {
    'AAPL': 150.00,
    'GOOGL': 2500.00,
    'MSFT': 300.00
};

// Create a simple HTTP server
const server = http.createServer((req, res) => {
    res.writeHead(200, { 'Content-Type': 'text/plain' });
    res.end('WebSocket Server is running');
});

// Create a WebSocket server attached to the HTTP server
const wss = new WebSocket.Server({ server });

// Function to simulate real-time stock price changes
function updateStockPrices() {
    for (const ticker in stocks) {
        // Generate a random small change
        const change = (Math.random() - 0.5) * 2; // -1 to +1
        stocks[ticker] = Math.round((stocks[ticker] + change) * 100) / 100;

        // Ensure price is not negative
        if (stocks[ticker] < 1) stocks[ticker] = 1.00;
    }

    // Prepare the message as a JSON string
    const message = JSON.stringify({
        type: 'STOCK_UPDATE',
        data: stocks,
        timestamp: new Date().toLocaleTimeString()
    });

    // Broadcast the updated prices to ALL connected clients
    wss.clients.forEach(client => {
        if (client.readyState === WebSocket.OPEN) {
            client.send(message);
        }
    });
}

// Start the continuous broadcasting every 1000ms (1 second)
setInterval(updateStockPrices, 1000);

// Event handling when a client connects
wss.on('connection', (ws) => {
    console.log('Client connected!');

    // Send initial data to the new client
    ws.send(JSON.stringify({
        type: 'INITIAL_DATA',
        data: stocks
    }));

    ws.on('close', () => {
        console.log('Client disconnected');
    });
});

const PORT = 8080;
server.listen(PORT, () => {
    console.log(`Server started on http://localhost:${PORT}`);
    console.log(`WebSocket server running on ws://localhost:${PORT}`);
});

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Real-Time Stock Ticker</title>
    <style>
        body { font-family: sans-serif; padding: 20px; }
        .ticker-item { margin: 10px 0; padding: 10px; border: 1px solid #ccc; border-left: 5px solid blue; }
        .price { font-size: 1.5em; font-weight: bold; }
        .up { color: green; }
        .down { color: red; }
    </style>
</head>
<body>
    <h1>Real-Time Stock Ticker</h1>
    <div id="ticker-container">
        <p>Connecting to server...</p>
    </div>
    <p>Last updated: <span id="timestamp">--</span></p>

    <script>
        const container = document.getElementById('ticker-container');
        const timestampEl = document.getElementById('timestamp');
        let previousPrices = {};

        // 1. Establish WebSocket connection
        const socket = new WebSocket('ws://localhost:8080'); 

        // 2. Handle connection events
        socket.onopen = () => {
            console.log('WebSocket connection established.');
        };

        socket.onerror = (error) => {
            console.error('WebSocket Error:', error);
            container.innerHTML = '<p style="color:red;">Error connecting to server. Make sure server.js is running.</p>';
        };

        // 3. Handle incoming messages (The Real-Time Data)
        socket.onmessage = (event) => {
            const message = JSON.parse(event.data);

            if (message.type === 'STOCK_UPDATE' || message.type === 'INITIAL_DATA') {
                updateTicker(message.data);
                timestampEl.textContent = message.timestamp;
            }
        };

        // 4. Function to update the UI
        function updateTicker(currentPrices) {
            let html = '';

            for (const [ticker, price] of Object.entries(currentPrices)) {
                const prevPrice = previousPrices[ticker] || price;
                let priceClass = '';
                
                if (price > prevPrice) {
                    priceClass = 'up'; // Price went up
                } else if (price < prevPrice) {
                    priceClass = 'down'; // Price went down
                }

                html += `
                    <div class="ticker-item">
                        <h2>${ticker}</h2>
                        <span class="price ${priceClass}">$${price.toFixed(2)}</span>
                    </div>
                `;
            }

            container.innerHTML = html;
            // Store current prices to compare in the next update
            previousPrices = currentPrices; 
        }

    </script>
</body>
</html>
