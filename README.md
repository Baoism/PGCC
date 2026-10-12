<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>TCG Pack Simulator</title>
    <style>
        body { font-family: sans-serif; background: #111; color: #fff; text-align: center; padding: 40px; }
        button { padding: 12px 24px; font-size: 16px; cursor: pointer; margin: 10px; background: #ffcc00; border: none; font-weight: bold; border-radius: 5px; }
        button:hover { background: #e6b800; }
        #log { margin-top: 20px; text-align: left; display: inline-block; background: #222; padding: 20px; border-radius: 8px; max-width: 450px; min-height: 100px; }
    </style>
</head>
<body>
    <h1>TCG Pack Simulator</h1>
    <p>Cash: $<span id="cash">1000</span></p>
    <button onclick="buyPack()">Buy Booster Pack ($100)</button>
    <div id="log"><b>Inventory Log:</b><br></div>

    <script>
        let cash = 1000;
        let inventory = [];

        function buyPack() {
            if (cash < 100) {
                alert('Not enough cash!');
                return;
            }
            cash -= 100;
            document.getElementById('cash').innerText = cash;

            let hits = ['Zubat (Bulk)', 'Pikachu (Common)', 'Charizard ex (Hit!)', 'Mewtwo SIR (Jackpot!)'];
            let card = hits[Math.floor(Math.random() * hits.length)];
            inventory.push(card);

            let logDiv = document.getElementById('log');
            logDiv.innerHTML = `Opened Pack! Pulled: <b>${card}</b><br>` + logDiv.innerHTML;
        }
    </script>
</body>
</html>
