# NWC Wallet Creation via Nostr DVM - Flow Diagram

## Overview
The `./create-nwc-wallet.sh` script creates Lightning wallets using the NIP-90 DVM (Data Vending Machine) protocol combined with NWC (Nostr Wallet Connect) standards.

## Architecture Flow Diagram

```mermaid
sequenceDiagram
    participant User as 👤 User
    participant Script as 📜 create-nwc-wallet.sh
    participant Python as 🐍 dvm-create-wallet.py
    participant NostrKeys as 🔐 NostrKeys
    participant Relay as 📡 Nostr Relay<br/>(ws://localhost:7000)
    participant Bridge as 🌉 NWC-Deposits Bridge<br/>(Docker Container)
    participant LND as ⚡ LND Deposits Vault
    participant WalletFile as 📁 Wallet File

    User->>Script: ./create-nwc-wallet.sh "My Wallet" 100000
    
    Note over Script: 🔍 Step 1: Environment Check
    Script->>Bridge: docker ps | grep deposits-nwc-deposits-bridge
    Bridge-->>Script: ✅ Bridge Running
    
    Script->>Bridge: docker logs (get bridge pubkey)
    Bridge-->>Script: 🔑 Bridge Pubkey: abc123...
    
    Note over Script: 🐍 Step 2: Python Environment Setup
    Script->>Script: Check/Create Python venv
    Script->>Script: source ../common/nwc-env/bin/activate
    Script->>Script: pip install requirements-nwc.txt
    
    Note over Script: 🚀 Step 3: DVM Wallet Creation
    Script->>Python: python3 ../common/dvm-create-wallet.py<br/>--bridge-pubkey abc123...<br/>--spending-limit 100000<br/>--wallet-name "My Wallet"
    
    Note over Python: 🔐 Key Generation
    Python->>NostrKeys: NostrKeys() # Generate client keys
    NostrKeys-->>Python: 🔑 Client Private/Public Key Pair
    
    Note over Python: 📡 Connect to Relay
    Python->>Relay: websockets.connect(ws://localhost:7000)
    Relay-->>Python: ✅ WebSocket Connection
    
    Python->>Relay: ["REQ", "dvm-client-abc", {"kinds": [6050-6059], "#p": [client_pubkey]}]
    Relay-->>Python: ✅ Subscribed to DVM Responses
    
    Note over Python: 📋 Create DVM Request
    Python->>Python: create_event(kind=5050,<br/>content={"wallet_name": "My Wallet", "spending_limit": 100000},<br/>tags=[["p", bridge_pubkey], ["t", "wallet-creation"]])
    
    Python->>NostrKeys: sign_event() # Schnorr signature
    NostrKeys-->>Python: ✅ Signed DVM Event
    
    Python->>Relay: ["EVENT", dvm_event]
    
    Note over Relay,Bridge: 🌉 DVM Processing
    Relay->>Bridge: Forward DVM Event (kind 5050)
    
    Note over Bridge: 🏗️ Wallet Creation Logic
    Bridge->>Bridge: Parse DVM request
    Bridge->>Bridge: Generate deposit keypair
    Bridge->>Bridge: Create spending limits config
    
    Bridge->>LND: Create new deposit account
    LND-->>Bridge: ✅ Deposit ID: d-xyz789
    
    Bridge->>Bridge: Generate NWC connection string<br/>nostrwalletconnect://bridge_pubkey?relay=ws://localhost:7000&secret=...
    
    Note over Bridge: 📤 DVM Response
    Bridge->>Bridge: create_event(kind=6050,<br/>content={"status": "success", "deposit_id": "d-xyz789", ...})
    Bridge->>Relay: ["EVENT", dvm_response]
    
    Relay->>Python: Forward DVM Response (kind 6050)
    
    Note over Python: ✅ Process Response
    Python->>Python: Parse DVM response
    Python->>WalletFile: Save wallet-d-xyz789.txt<br/>NWC_CONNECTION=nostrwalletconnect://...<br/>DEPOSIT_ID=d-xyz789<br/>DEPOSIT_PUBKEY=03abc...<br/>SPENDING_LIMIT=100000
    
    Python-->>Script: ✅ Wallet creation success<br/>💾 Connection saved to: wallet-d-xyz789.txt
    
    Note over Script: 📋 Result Processing
    Script->>WalletFile: Extract NWC_CONNECTION, DEPOSIT_ID, DEPOSIT_PUBKEY
    Script->>Script: Fix relay URL (nostr-relay:8080 → localhost:7000)
    Script->>WalletFile: Create clean wallet file with usage instructions
    
    Script-->>User: 🎉 Wallet Created!<br/>📋 Next Steps:<br/>./nwc-client.sh --connection "..." --quick-test<br/>📱 Or copy to wallet app (Alby, Zeus, Mutiny)

    Note over User,LND: 🔄 Ready for Operations
    Note over User: User can now:<br/>💳 Fund wallet via Lightning invoice<br/>⚡ Send payments via NWC<br/>📱 Connect to mobile apps<br/>🔍 Check balance and info
```

## Technical Details

### 🔐 Cryptographic Components

1. **Nostr Keys (Client)**
   - **Type**: Schnorr signatures (BIP-340)
   - **Purpose**: Sign DVM requests to Nostr relay
   - **Format**: x-only public keys (32 bytes)

2. **Deposit Keys (Backend)**
   - **Type**: ECDSA signatures
   - **Purpose**: Authenticate with LND Deposits API
   - **Format**: Compressed public keys (33 bytes)

3. **Bridge Keys**
   - **Type**: Schnorr signatures
   - **Purpose**: Bridge identity and DVM responses
   - **Format**: x-only public keys (32 bytes)

### 📡 Protocol Stack

| Layer | Protocol | Purpose |
|-------|----------|---------|
| Application | Shell Script | User interface and orchestration |
| DVM Client | NIP-90 | Wallet creation requests (kinds 5050-5059) |
| NWC Operations | NIP-47 | Wallet operations (kinds 23194-23195) |
| Transport | WebSocket | Real-time communication with relay |
| Backend | LND Deposits API | Actual Lightning wallet functionality |

### 🌉 Bridge Architecture

```
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│   NWC Client    │───▶│  DVM+NWC Bridge  │───▶│  LND Deposits   │
│                 │    │                  │    │      Vault      │
│ • Nostr Events  │    │ • DVM Processing │    │ • Real Lightning│
│ • NWC Requests  │    │ • Protocol Trans │    │ • Deposit Mgmt  │
│ • Wallet Apps   │    │ • Key Management │    │ • Balance Track │
└─────────────────┘    └──────────────────┘    └─────────────────┘
```

### 🎯 Key Features

- **🤖 Automated**: Single command creates complete Lightning wallet
- **🔐 Secure**: Multi-layer cryptographic authentication
- **📱 Compatible**: Works with all NWC-supporting wallet apps
- **⚡ Zero-UTXO**: No on-chain setup required
- **🌐 Decentralized**: Uses Nostr relay for communication
- **🧪 Testable**: Includes built-in testing tools

### 📁 File Outputs

The script creates multiple files for different use cases:

1. **Original DVM File**: `wallet-{timestamp}.txt`
   - Raw output from DVM protocol
   - Docker-internal relay URLs

2. **Clean Wallet File**: `wallet_dvm_{timestamp}.txt`
   - Host-accessible relay URLs
   - Usage instructions included
   - Ready for production use

3. **Connection String**: For direct wallet app integration
   ```
   nostrwalletconnect://bridge_pubkey?relay=ws://localhost:7000&secret=...
   ```

## Next Steps After Wallet Creation

1. **🧪 Quick Test**: `./nwc-client.sh --connection "..." --quick-test`
2. **📱 Mobile Apps**: Copy connection string to Alby, Zeus, or Mutiny
3. **💳 Fund Wallet**: Create invoice and receive Lightning payments
4. **⚡ Send Payments**: Pay Lightning invoices via NWC protocol
5. **🔄 Interactive Mode**: `./nwc-client.sh --connection "..." --interactive`

This architecture enables instant Lightning wallet creation with zero on-chain setup, combining the best of Nostr's decentralized communication with Lightning's payment capabilities.