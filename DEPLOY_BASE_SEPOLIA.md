# Stix auf Base Sepolia deployen — Schritt für Schritt

Ziel: Den Contract auf dem **Testnet** (Base Sepolia) zum Laufen bringen. Dort kostet alles
nur Test-ETH, das du gratis bekommst. Plane ~30–45 Min ein.

Reihenfolge:
1. Tools installieren (Foundry)
2. Projekt vorbereiten + Dependencies
3. Wallet & Test-ETH vorbereiten
4. Chainlink VRF-Subscription anlegen
5. Contract deployen
6. Contract als VRF-Consumer registrieren
7. Mint öffnen & testen

---

## Base-Sepolia-Werte (von Chainlink, frisch geprüft)

| Item | Wert |
|---|---|
| VRF Coordinator | `0x5C210eF41CD1a72de73bF76eC39637bB0d3d7BEE` |
| Key Hash (30 gwei) | `0x9e1344a1247c8a1785d0a4681a27152bffdb43666ae5bf7d14d24a5efd44bf71` |
| LINK Token | `0xE4aB69C077896252FAFBD49EFD26B5D171A32410` |
| Chain ID | `84532` (hex `0x14a34`) |
| RPC URL | `https://sepolia.base.org` |
| Explorer | `https://sepolia.basescan.org` |

> Diese Adressen ändern sich gelegentlich. Wenn etwas nicht klappt, vergleiche sie mit
> https://docs.chain.link/vrf/v2-5/supported-networks (Abschnitt „BASE Sepolia Testnet").

---

## Schritt 1 — Foundry installieren

Foundry ist das Toolkit zum Bauen/Deployen. Im Terminal (Mac):

```bash
curl -L https://foundry.paradigm.xyz | bash
```

Terminal neu starten (oder `source ~/.zshenv` / `source ~/.bashrc`), dann:

```bash
foundryup
```

Prüfen, dass es da ist:

```bash
forge --version
```

---

## Schritt 2 — Projekt vorbereiten

Du hast den Ordner `stix-contracts` mit den `.sol`-Dateien. Geh hinein:

```bash
cd stix-contracts
```

Falls dort noch kein Foundry-Projekt initialisiert ist, einmalig:

```bash
forge init --force --no-commit
```

(Das `--force` behält deine bestehenden `src/`-Dateien.)

Dann die Dependencies installieren:

```bash
forge install foundry-rs/forge-std --no-commit
forge install OpenZeppelin/openzeppelin-contracts --no-commit
forge install smartcontractkit/chainlink-brownie-contracts --no-commit
```

Kompilieren als Test:

```bash
forge build
```

Wenn es Importpfad-Fehler gibt, prüfe, dass `foundry.toml` die `remappings` enthält (sind
in der mitgelieferten Datei schon drin).

Tests laufen lassen (lokal, kein Netzwerk nötig):

```bash
forge test -vvv
```

---

## Schritt 3 — Wallet & Test-ETH

Du brauchst eine Wallet (z.B. MetaMask) mit **Base Sepolia Test-ETH**.

1. Base Sepolia zu MetaMask hinzufügen (falls noch nicht): Netzwerk mit obiger RPC-URL,
   Chain ID `84532`, Symbol `ETH`, Explorer `https://sepolia.basescan.org`.
2. Test-ETH holen — eine dieser Quellen:
   - https://www.alchemy.com/faucets/base-sepolia
   - https://faucet.quicknode.com/base/sepolia
   - Coinbase Wallet Faucet (in der App)
3. Du brauchst auch **Test-LINK** für die VRF-Subscription:
   - https://faucets.chain.link/base-sepolia  (gibt LINK + etwas ETH)

Exportiere deinen **Private Key** aus MetaMask (Account Details → Export Private Key).
**Niemals** den Private Key einer Wallet mit echtem Geld benutzen — nimm eine reine Testnet-Wallet.

---

## Schritt 4 — VRF-Subscription anlegen

1. Geh auf https://vrf.chain.link
2. Wallet verbinden, oben **Base Sepolia** als Netzwerk wählen.
3. **Create Subscription** → bestätigen (kostet etwas Test-ETH Gas).
4. Du bekommst eine **Subscription ID** (eine lange Zahl). Notiere sie.
5. **Fund subscription** → ein paar Test-LINK einzahlen (z.B. 5–10 LINK reichen für viele Tests).

Den Consumer (= deinen Contract) fügst du erst in Schritt 6 hinzu, weil er noch nicht
deployt ist.

---

## Schritt 5 — Contract deployen

Lege im `stix-contracts`-Ordner eine Datei `.env` an (wird nicht hochgeladen):

```bash
BASE_SEPOLIA_RPC_URL=https://sepolia.base.org
BASESCAN_API_KEY=dein_basescan_api_key
PRIVATE_KEY=0xDEIN_TESTNET_PRIVATE_KEY
VRF_COORDINATOR=0x5C210eF41CD1a72de73bF76eC39637bB0d3d7BEE
VRF_SUB_ID=DEINE_SUBSCRIPTION_ID
VRF_KEY_HASH=0x9e1344a1247c8a1785d0a4681a27152bffdb43666ae5bf7d14d24a5efd44bf71
```

> Den BaseScan API Key bekommst du gratis auf https://basescan.org (Account → API Keys).
> Er ist nur für die Verifizierung nötig; zum reinen Deployen kannst du ihn weglassen.

Env-Variablen laden und deployen:

```bash
source .env

forge script script/Deploy.s.sol \
  --rpc-url $BASE_SEPOLIA_RPC_URL \
  --broadcast \
  --verify \
  --etherscan-api-key $BASESCAN_API_KEY
```

Am Ende druckt das Script zwei Adressen:
- `StixRenderer: 0x...`
- `Stix: 0x...`

**Notiere dir die `Stix`-Adresse** — die brauchst du gleich und später im Frontend.

(Falls `--verify` zickt, lass es zunächst weg und verifiziere später manuell. Deployen geht
auch ohne Verifizierung.)

---

## Schritt 6 — Contract als VRF-Consumer registrieren

Damit dein Contract die Subscription nutzen darf:

1. Zurück auf https://vrf.chain.link → deine Subscription öffnen.
2. **Add consumer** → die **Stix-Contract-Adresse** aus Schritt 5 einfügen → bestätigen.

Jetzt darf der Contract Randomness anfordern und die Subscription zahlt dafür mit LINK.

---

## Schritt 7 — Mint öffnen & testen

Der Mint ist standardmäßig zu. Öffne ihn. Schnellster Weg ist `cast` (kommt mit Foundry):

```bash
source .env

# Mint öffnen
cast send <STIX_ADRESSE> "setMintOpen(bool)" true \
  --rpc-url $BASE_SEPOLIA_RPC_URL --private-key $PRIVATE_KEY

# Optional: Preis auf 0 setzen fürs Testen (oder einen kleinen Wert)
cast send <STIX_ADRESSE> "setPrice(uint256)" 0 \
  --rpc-url $BASE_SEPOLIA_RPC_URL --private-key $PRIVATE_KEY
```

Test-Mint auslösen (bei Preis 0 ohne `--value`):

```bash
cast send <STIX_ADRESSE> "requestMint()" \
  --rpc-url $BASE_SEPOLIA_RPC_URL --private-key $PRIVATE_KEY
```

Nach ein paar Sekunden liefert Chainlink die Randomness und der Token wird gemintet. Prüfen:

```bash
# Wie viele Tokens existieren / Besitzer von Token 1
cast call <STIX_ADRESSE> "ownerOf(uint256)" 1 --rpc-url $BASE_SEPOLIA_RPC_URL

# tokenURI ansehen (gibt das base64-JSON mit dem SVG zurück)
cast call <STIX_ADRESSE> "tokenURI(uint256)" 1 --rpc-url $BASE_SEPOLIA_RPC_URL
```

Den Token siehst du auch auf https://sepolia.basescan.org unter deiner Contract-Adresse,
und (etwas später) im Testnet-OpenSea.

---

## Häufige Stolpersteine

- **„InsufficientBalance" bei requestMint:** Die VRF-Subscription hat kein LINK mehr → nachfüllen.
- **Mint-Tx geht durch, aber kein Token erscheint:** Der VRF-Callback braucht ein paar
  Sekunden. Wenn nach ~1 Min nichts kommt: Consumer in der Subscription korrekt hinzugefügt?
  Genug LINK? `vrfCallbackGasLimit` hoch genug (Standard 200000 reicht für 3 Sticks)?
- **„WrongPrice":** Du musst exakt den `price`-Wert als `--value` mitschicken. Bei Preis 0
  ohne `--value`.
- **Verify schlägt fehl:** Nicht schlimm fürs Testen — Funktionalität ist unabhängig davon.

---

## Danach: Frontend verbinden

Sobald der Contract auf Base Sepolia läuft, bauen wir die echten Calls ins Frontend ein:
`requestMint`, `mergeBurn`, `saveLayout`, und das Lesen deiner NFTs über `tokenURI`. Dann
testest du die ganze App end-to-end gegen das Testnet. Sag Bescheid, wenn du hier bist.
