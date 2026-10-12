<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>TCG Sourcing & Market Simulator</title>
    <style>
        * { box-sizing: border-box; font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif; }
        body { background: #121214; color: #f0f0f0; margin: 0; padding: 20px; }
        h1, h2, h3 { margin-top: 0; }
        .header { display: flex; justify-content: space-between; align-items: center; background: #1e1e24; padding: 15px 25px; border-radius: 12px; margin-bottom: 20px; border: 1px solid #333; }
        .cash { font-size: 24px; color: #4CAF50; font-weight: bold; }
        .net-worth { font-size: 18px; color: #ffd700; }
        
        .grid { display: grid; grid-template-columns: 1fr 1fr; gap: 20px; }
        @media(max-width: 900px) { .grid { grid-template-columns: 1fr; } }

        .card-panel { background: #1e1e24; padding: 20px; border-radius: 12px; border: 1px solid #333; margin-bottom: 20px; }
        button { background: #ffcc00; color: #000; border: none; padding: 10px 18px; font-weight: bold; border-radius: 6px; cursor: pointer; transition: 0.2s; }
        button:hover { background: #e6b800; transform: translateY(-1px); }
        button:disabled { background: #555; color: #888; cursor: not-allowed; transform: none; }
        
        .market-item { display: flex; justify-content: space-between; align-items: center; background: #2a2a32; padding: 12px; border-radius: 8px; margin-bottom: 10px; }
        .card-item { background: #2a2a32; padding: 12px; border-radius: 8px; margin-bottom: 10px; border-left: 4px solid #555; display: flex; justify-content: space-between; align-items: center; }
        
        .rarity-Common { border-color: #888; }
        .rarity-Uncommon { border-color: #4CAF50; }
        .rarity-Rare { border-color: #2196F3; }
        .rarity-Bulk { border-color: #ff9800; }
        .rarity-Filler { border-color: #9c27b0; }
        .rarity-HeavyHit { border-color: #e91e63; }
        .rarity-Chase { border-color: #ffd700; background: #3b3112; box-shadow: 0 0 15px rgba(255, 215, 0, 0.3); }

        .tag { font-size: 11px; padding: 3px 6px; border-radius: 4px; background: #111; margin-left: 5px; text-transform: uppercase; }
        .tag-sealed { color: #00bcd4; }
        .tag-raw { color: #ff9800; }
        .tag-graded { color: #ffd700; }

        /* Animation styling for pack opening */
        .pack-rip-area { text-align: center; background: #18181c; padding: 20px; border-radius: 10px; border: 2px dashed #444; margin-top: 15px; }
        .card-reveal-box {
            perspective: 1000px;
            margin: 20px auto;
            width: 220px;
            height: 310px;
        }
        .card-inner {
            position: relative;
            width: 100%;
            height: 100%;
            text-align: center;
            transition: transform 0.6s;
            transform-style: preserve-3d;
            border-radius: 12px;
        }
        .card-inner.flipped {
            transform: rotateY(180deg);
        }
        .card-front, .card-back {
            position: absolute;
            width: 100%;
            height: 100%;
            backface-visibility: hidden;
            display: flex;
            flex-direction: column;
            justify-content: center;
            align-items: center;
            border-radius: 12px;
            padding: 15px;
            border: 2px solid #444;
        }
        .card-back {
            background: linear-gradient(135deg, #2b2b36, #1a1a22);
            color: #ffcc00;
            font-size: 20px;
            font-weight: bold;
        }
        .card-front {
            background: #25252e;
            color: #fff;
            transform: rotateY(180deg);
        }
    </style>
</head>
<body>

    <div class="header">
        <div>
            <h1>TCG Sourcing & Market Simulator</h1>
            <div style="font-size: 12px; color: #4CAF50;">✨ Pack Animation Module Active</div>
        </div>
        <div>
            <div class="cash">Cash: $<span id="cash">1000.00</span></div>
            <div class="net-worth">Portfolio Net Worth: $<span id="netWorth">1000.00</span></div>
        </div>
    </div>

    <div class="grid">
        <!-- Shop / Sourcing -->
        <div>
            <div class="card-panel">
                <h2>🏪 1. The Shop (Sourcing)</h2>
                <div id="shopList"></div>
            </div>

            <div class="card-panel">
                <h2>✂️ 2. Sealed Inventory & Pack Ripping</h2>
                <div id="sealedInventory"></div>
                <div id="packRipArea" class="pack-rip-area" style="display:none;">
                    <h3 id="packProgressTitle">Opening Pack...</h3>
                    
                    <div class="card-reveal-box">
                        <div id="cardInner" class="card-inner">
                            <div class="card-back">TCG PACK<br><span style="font-size: 12px; color: #aaa; margin-top: 10px;">Click Flip</span></div>
                            <div id="cardFront" class="card-front">---</div>
                        </div>
                    </div>

                    <button id="actionCardBtn" onclick="handleCardAction()">Flip Card</button>
                </div>
            </div>
        </div>

        <!-- Portfolio & Grading -->
        <div>
            <div class="card-panel">
                <h2>💼 3. Collection & Market Vault</h2>
                <div id="cardInventory"></div>
            </div>

            <div class="card-panel">
                <h2>🔬 4. Grading Service (PSA/CGC/Beckett)</h2>
                <p style="font-size: 13px; color: #aaa;">Submit raw cards to grade. Increases resale value based on sub-scores!</p>
                <div id="gradingQueue"></div>
            </div>
        </div>
    </div>

    <script>
        let cash = 1000.00;
        let inventory = [];
        let sealedVault = [];
        let currentPackCards = [];
        let currentCardIndex = 0;
        let isFlipped = false;

        const COMMONS = ["Pidgey", "Rattata", "Caterpie", "Weedle", "Metapod", "Kakuna", "Spearow", "Ekans", "Oddish", "Tentacool", "Zubat", "Geodude", "Magnemite"];
        const UNCOMMONS = ["Gloom", "Weepinbell", "Golbat", "Graveler", "Magneton", "Poliwhirl", "Haunter", "Kadabra", "Machoke"];
        const TIER_1_BULK = ["Zubat (Holo)", "Yungoos (Holo)", "Greedent (V)", "Patrat (Holo)", "Bidoof (Meme Special)", "Trubbish (Holo)", "Garbodor (ex)", "Bruxish (ex)", "Spidops (Tera ex)"];
        const TIER_2_FILLER = ["Chikorita (IR)", "Cyndaquil (IR)", "Totodile (IR)", "Treecko (Holo)", "Torchic (Holo)", "Mareep (GG Art)", "Wooper (Shiny)", "Marill (IR)", "Psyduck (IR)"];
        const TIER_3_HEAVY = ["Gengar (ex)", "Alakazam (ex)", "Machamp (VMAX)", "Gyarados (Tera ex)", "Lapras (ex)", "Snorlax (Secret)", "Dragonite (ex)", "Tyranitar (Tera ex)", "Greninja (Tera ex)"];
        const TIER_4_CHASE = ["Tag Team (Mewtwo & Mew)", "Giratina (Alt Art)", "Roaring Moon (SIR)", "Lugia (Alt Art)", "Rayquaza (Gold Star)", "Charizard (Base Holo)", "Pikachu (Stellar Tera)"];

        const PRODUCTS = [
            { id: 'booster', name: 'Booster Pack', price: 10, packs: 1, sentiment: '+1.2%' },
            { id: 'blister', name: '3-Pack Blister', price: 30, packs: 3, sentiment: '+2.5%' },
            { id: 'etb', name: 'Elite Trainer Box (ETB)', price: 80, packs: 8, sentiment: '+4.2%' },
            { id: 'box', name: 'Booster Box (36 Packs)', price: 320, packs: 36, sentiment: '+0.8%' }
        ];

        function updateUI() {
            document.getElementById('cash').innerText = cash.toFixed(2);
            let cardValue = inventory.reduce((acc, card) => acc + getItemMarketValue(card), 0);
            let sealedValue = sealedVault.reduce((acc, item) => acc + item.price, 0);
            document.getElementById('netWorth').innerText = (cash + cardValue + sealedValue).toFixed(2);

            renderShop();
            renderSealed();
            renderInventory();
        }

        function getItemMarketValue(card) {
            let base = card.baseValue;
            if (card.state === 'Graded') {
                return base * getGradedMultiplier(card.gradingInfo.finalGrade, card.gradingInfo.isBlackLabel);
            }
            return base;
        }

        function getGradedMultiplier(grade, isBlackLabel) {
            if (isBlackLabel) return 25.0;
            if (grade === 10) return 4.5;
            if (grade === 9.5) return 2.2;
            if (grade === 9.0) return 1.4;
            if (grade === 8.5) return 1.1;
            if (grade === 8.0) return 0.9;
            return 0.5;
        }

        function renderShop() {
            let html = '';
            PRODUCTS.forEach(p => {
                html += `
                    <div class="market-item">
                        <div>
                            <strong>${p.name}</strong> <span class="badge">${p.sentiment}</span>
                            <div style="font-size:12px; color:#aaa;">Contains ${p.packs} Pack(s)</div>
                        </div>
                        <div>
                            <button onclick="buyProduct('${p.id}')">Buy $${p.price}</button>
                        </div>
                    </div>`;
            });
            document.getElementById('shopList').innerHTML = html;
        }

        function buyProduct(prodId) {
            let p = PRODUCTS.find(prod => prod.id === prodId);
            if (cash < p.price) { alert("Not enough cash!"); return; }
            cash -= p.price;
            sealedVault.push({ ...p, id: Date.now() + Math.random() });
            updateUI();
        }

        function renderSealed() {
            let html = '';
            if (sealedVault.length === 0) html = '<p style="font-size:13px; color:#888;">No sealed product in vault.</p>';
            sealedVault.forEach(item => {
                html += `
                    <div class="market-item">
                        <div><strong>${item.name}</strong> <span class="tag tag-sealed">Sealed</span></div>
                        <div>
                            <button onclick="ripProduct(${item.id})">Rip Now ✂️</button>
                        </div>
                    </div>`;
            });
            document.getElementById('sealedInventory').innerHTML = html;
        }

        function ripProduct(id) {
            let index = sealedVault.findIndex(item => item.id === id);
            if (index === -1) return;
            let item = sealedVault.splice(index, 1)[0];
            
            currentPackCards = [];
            for (let i = 0; i < item.packs; i++) {
                currentPackCards.push(...generateSinglePack());
            }

            currentCardIndex = 0;
            document.getElementById('packRipArea').style.display = 'block';
            prepNextCardForAnimation();
            updateUI();
        }

        function generateSinglePack() {
            let pack = [];
            for (let i = 0; i < 4; i++) {
                let name = COMMONS[Math.floor(Math.random() * COMMONS.length)];
                pack.push(createCard(name, 'Common', 0.50));
            }
            for (let i = 0; i < 3; i++) {
                let name = UNCOMMONS[Math.floor(Math.random() * UNCOMMONS.length)];
                pack.push(createCard(name, 'Uncommon', 1.50));
            }
            pack.push(createCard("Reverse Holo " + COMMONS[0], 'Rare', 3.00));
            pack.push(rollHitSlot());
            pack.push(rollHitSlot());
            return pack;
        }

        function rollHitSlot() {
            let roll = Math.random() * 100;
            if (roll <= 5) {
                let name = TIER_4_CHASE[Math.floor(Math.random() * TIER_4_CHASE.length)];
                return createCard(name, 'Chase', 120.00);
            } else if (roll <= 20) {
                let name = TIER_3_HEAVY[Math.floor(Math.random() * TIER_3_HEAVY.length)];
                return createCard(name, 'HeavyHit', 35.00);
            } else if (roll <= 50) {
                let name = TIER_2_FILLER[Math.floor(Math.random() * TIER_2_FILLER.length)];
                return createCard(name, 'Filler', 8.00);
            } else {
                let name = TIER_1_BULK[Math.floor(Math.random() * TIER_1_BULK.length)];
                return createCard(name, 'Bulk', 2.00);
            }
        }

        function createCard(name, tier, baseVal) {
            return {
                id: Date.now() + Math.random(),
                name: name,
                tier: tier,
                baseValue: baseVal,
                state: 'Raw',
                conditionSeed: {
                    centering: Math.random() * (10 - 7) + 7,
                    corners: Math.random() * (10 - 7) + 7,
                    edges: Math.random() * (10 - 7) + 7,
                    surface: Math.random() * (10 - 7) + 7
                }
            };
        }

        function prepNextCardForAnimation() {
            isFlipped = false;
            document.getElementById('cardInner').classList.remove('flipped');
            
            let card = currentPackCards[currentCardIndex];
            let frontHTML = `
                <div style="font-size:12px; color:#aaa; text-transform:uppercase;">${card.tier}</div>
                <div style="font-size:16px; font-weight:bold; margin: 10px 0;">${card.name}</div>
                <div style="font-size:14px; color:#4CAF50;">$${card.baseValue.toFixed(2)}</div>
            `;
            document.getElementById('cardFront').innerHTML = frontHTML;
            document.getElementById('cardFront').className = `card-front rarity-${card.tier}`;
            document.getElementById('packProgressTitle').innerText = `Card ${currentCardIndex + 1} of ${currentPackCards.length}`;
            document.getElementById('actionCardBtn').innerText = "Flip Card ✨";
        }

        function handleCardAction() {
            let inner = document.getElementById('cardInner');
            let btn = document.getElementById('actionCardBtn');

            if (!isFlipped) {
                // Flip card face up
                inner.classList.add('flipped');
                inventory.push(currentPackCards[currentCardIndex]);
                isFlipped = true;
                
                if (currentCardIndex < currentPackCards.length - 1) {
                    btn.innerText = "Next Card ➡️";
                } else {
                    btn.innerText = "Finish Opening 🎉";
                }
                updateUI();
            } else {
                // Move to next card or close
                currentCardIndex++;
                if (currentCardIndex < currentPackCards.length) {
                    prepNextCardForAnimation();
                } else {
                    document.getElementById('packRipArea').style.display = 'none';
                }
            }
        }

        function renderInventory() {
            let html = '';
            if (inventory.length === 0) html = '<p style="font-size:13px; color:#888;">No cards in collection.</p>';
            inventory.forEach(card => {
                let val = getItemMarketValue(card);
                let stateTag = card.state === 'Graded' 
                    ? `<span class="tag tag-graded">${card.gradingInfo.company} ${card.gradingInfo.finalGrade}${card.gradingInfo.isBlackLabel ? ' (BLACK LABEL)' : ''}</span>`
                    : `<span class="tag tag-raw">Raw</span>`;

                html += `
                    <div class="card-item rarity-${card.tier}">
                        <div>
                            <strong>${card.name}</strong> ${stateTag}
                            <div style="font-size:12px; color:#aaa;">Market Value: $${val.toFixed(2)}</div>
                        </div>
                        <div>
                            ${card.state === 'Raw' ? `<button onclick="gradeCard('${card.id}')" style="font-size:11px;">Grade ($15)</button>` : ''}
                            <button onclick="sellCard('${card.id}')" style="font-size:11px; background:#4CAF50; color:#fff;">Sell</button>
                        </div>
                    </div>`;
            });
            document.getElementById('cardInventory').innerHTML = html;
        }

        function sellCard(cardId) {
            let idx = inventory.findIndex(c => c.id == cardId);
            if (idx === -1) return;
            let card = inventory[idx];
            
            let sellChance = card.state === 'Graded' ? 0.85 : 0.50;
            if (Math.random() <= sellChance) {
                cash += getItemMarketValue(card);
                inventory.splice(idx, 1);
                updateUI();
            } else {
                alert("No buyer found on marketplace right now. Try again!");
            }
        }

        function gradeCard(cardId) {
            let card = inventory.find(c => c.id == cardId);
            if (!card) return;
            if (cash < 15) { alert("Not enough cash to grade ($15 fee)!"); return; }

            cash -= 15;
            let company = Math.random() > 0.8 ? 'Beckett' : 'PSA';
            let seed = card.conditionSeed;
            let lowest = Math.min(seed.centering, seed.corners, seed.edges, seed.surface);
            let avg = (seed.centering + seed.corners + seed.edges + seed.surface) / 4;
            
            let finalGrade = Math.min(10.0, Math.min(Math.floor(avg * 2) / 2, lowest + 0.5));
            let isBlackLabel = (company === 'Beckett' && finalGrade === 10 && Math.random() <= 0.03);

            card.state = 'Graded';
            card.gradingInfo = {
                company: company,
                finalGrade: finalGrade,
                isBlackLabel: isBlackLabel
            };

            updateUI();
        }

        updateUI();
    </script>
</body>
</html>
