<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>TCG Simulator & Market Loop</title>
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
        .rarity-Chase { border-color: #ffd700; background: #3b3112; }

        .tag { font-size: 11px; padding: 3px 6px; border-radius: 4px; background: #111; margin-left: 5px; text-transform: uppercase; }
        .tag-sealed { color: #00bcd4; }
        .tag-raw { color: #ff9800; }
        .tag-graded { color: #ffd700; }

        .pack-rip-area { text-align: center; background: #18181c; padding: 20px; border-radius: 10px; border: 2px dashed #444; margin-top: 15px; }
        .revealed-card { font-size: 18px; font-weight: bold; margin: 10px 0; padding: 15px; background: #25252e; border-radius: 8px; }
        
        .badge { display: inline-block; padding: 2px 8px; border-radius: 12px; font-size: 12px; font-weight: bold; background: #333; }
        .warning-text { color: #ff5555; font-size: 12px; font-style: italic; margin-top: 4px; }
    </style>
</head>
<body>

    <div class="header">
        <div>
            <h1>TCG Sourcing & Market Simulator</h1>
            <div class="warning-text">⚠️ All packs give the same cards until updating</div>
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
                    <h3>Opening Pack...</h3>
                    <div id="cardDisplay" class="revealed-card">---</div>
                    <button id="nextCardBtn" onclick="nextCard()">Next Card (10 Remaining)</button>
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
        // --- GAME STATE ---
        let cash = 1000.00;
        let inventory = []; // Raw/Graded Cards
        let sealedVault = []; // Sealed Products
        let currentPackCards = [];
        let currentCardIndex = 0;

        // --- DATA TABLES ---
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

        // --- CORE FUNCTIONS ---
        function updateUI() {
            document.getElementById('cash').innerText = cash.toFixed(2);
            
            // Calculate Portfolio Net Worth
            let cardValue = inventory.reduce((acc, card) => acc + getItemMarketValue(card), 0);
            let sealedValue = sealedVault.reduce((acc, item) => acc + item.price, 0);
            document.getElementById('netWorth').innerText = (cash + cardValue + sealedValue).toFixed(2);

            renderShop();
            renderSealed();
            renderInventory();
            renderGrading();
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

        // --- SHOP & SOURCING LOOP ---
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
            sealedVault.push({ ...p, id: Date.now() });
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

        // --- PACK ENGINE & PULL RATES ---
        function ripProduct(id) {
            let index = sealedVault.findIndex(item => item.id === id);
            if (index === -1) return;
            let item = sealedVault.splice(index, 1)[0];
            
            // Build cards for all packs in item
            currentPackCards = [];
            for (let i = 0; i < item.packs; i++) {
                currentPackCards.push(...generateSinglePack());
            }

            currentCardIndex = 0;
            document.getElementById('packRipArea').style.display = 'block';
            updatePackDisplay();
            updateUI();
        }

        function generateSinglePack() {
            let pack = [];
            // Slots 1-4: Commons
            for (let i = 0; i < 4; i++) {
                let name = COMMONS[Math.floor(Math.random() * COMMONS.length)];
                pack.push(createCard(name, 'Common', 0.50));
            }
            // Slots 5-7: Uncommons
            for (let i = 0; i < 3; i++) {
                let name = UNCOMMONS[Math.floor(Math.random() * UNCOMMONS.length)];
                pack.push(createCard(name, 'Uncommon', 1.50));
            }
            // Slot 8: Reverse Holo
            pack.push(createCard("Reverse Holo " + COMMONS[0], 'Rare', 3.00));

            // Slot 9 & 10: Hits
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

        function nextCard() {
            if (currentCardIndex < currentPackCards.length) {
                inventory.push(currentPackCards[currentCardIndex]);
                currentCardIndex++;
                updatePackDisplay();
                updateUI();
            }
        }

        function updatePackDisplay() {
            let btn = document.getElementById('nextCardBtn');
            let disp = document.getElementById('cardDisplay');

            if (currentCardIndex < currentPackCards.length) {
                let card = currentPackCards[currentCardIndex];
                disp.innerText = `${card.name} [${card.tier}] - Est. Raw: $${card.baseValue.toFixed(2)}`;
                btn.innerText = `Keep & Next Card (${currentPackCards.length - currentCardIndex - 1} Remaining)`;
            } else {
                disp.innerText = "✨ Pack Opening Complete!";
                btn.innerText = "Done";
                btn.onclick = () => {
                    document.getElementById('packRipArea').style.display = 'none';
                    btn.onclick = nextCard;
                };
            }
        }

        // --- INVENTORY & MARKETPLACE ---
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
            
            // AI Collector probability check
            let sellChance = card.state === 'Graded' ? 0.85 : 0.50;
            if (Math.random() <= sellChance) {
                cash += getItemMarketValue(card);
                inventory.splice(idx, 1);
                updateUI();
            } else {
                alert("No buyer found on marketplace right now. Try again!");
            }
        }

        // --- GRADING SIMULATOR (PSA / BECKETT ALGORITHM) ---
        function gradeCard(cardId) {
            let card = inventory.find(c => c.id == cardId);
            if (!card) return;
            if (cash < 15) { alert("Not enough cash to grade ($15 fee)!"); return; }

            cash -= 15;
            
            // Sub-score grading calculation engine
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

        function renderGrading() {
            // Placeholder for active turnaround timers
            document.getElementById('gradingQueue').innerHTML = `<p style="font-size:12px; color:#888;">Grading Queue Empty. Direct submission returned instantly!</p>`;
        }

        // Initialize App
        updateUI();
    </script>
</body>
</html>
