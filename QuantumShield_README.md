# QuantumShield

**Post-Quantum, Blockchain-Anchored Filesystem Audit Trail**

QuantumShield is a file integrity monitoring and forensic logging system. It does not try to prevent every file change. Instead, it creates a trusted, signed, blockchain-backed record of what changed, when it changed, and how the file state evolved over time.

The system is designed to support post-compromise investigation: even if an attacker changes a watched file and later restores it to look clean, QuantumShield preserves the historical evidence.

---

## 1. What QuantumShield Does

QuantumShield monitors selected filesystem paths and records file lifecycle events such as:

- `CREATED`
- `MODIFIED`
- `DELETED`
- `RENAMED`
- `MOVED`
- `MOVED_RENAMED`

Each event is:

1. detected by the watcher,
2. canonicalized as JSON,
3. signed by the post-quantum signer using **ML-DSA-65**,
4. submitted through the QS API,
5. verified and stored on **Hyperledger Fabric**,
6. displayed through the dashboard.

The key value of the system is not prevention. The key value is **immutable forensic evidence**.

---

## 2. Main Components

Expected repository layout:

```text
QuantumShield/
├── blockchain/
│   ├── docker-compose.yml
│   ├── chaincode/
│   │   └── pqcCC_demo/
│   ├── configtx/
│   ├── qs-api/
│   └── dashboard/
│       ├── backend/
│       ├── frontend/
│       └── docker-compose.yml
│
└── qs-mtls-demo/
    ├── docker-compose.yml
    ├── pqc-signer/
    ├── watcher/
    ├── mtls/
    ├── watched_folder/
    └── watcher_data/
```

### Component roles

| Component | Purpose |
|---|---|
| `watcher` | Watches files and generates canonical file events |
| `pqc-signer` | Signs events using ML-DSA-65 over mTLS |
| `qs-api` | Receives signed events and submits them to Fabric chaincode |
| `pqcCC_demo_final` | Fabric chaincode that verifies signatures and stores signed events |
| `fim-api` | Dashboard backend, Fabric query layer, log/alert collector |
| `fim-ui` | Web dashboard UI |
| Hyperledger Fabric network | Immutable ledger layer |

---

## 3. Requirements

Install these before running the project:

- Linux environment recommended
- Docker
- Docker Compose v2
- `curl`
- `jq`
- `openssl`
- `sha256sum`

Check them:

```bash
docker --version
docker compose version
curl --version
jq --version
openssl version
sha256sum --version
```

Make sure Docker is running:

```bash
docker ps
```

---

## 4. Create Docker Networks

QuantumShield uses two shared Docker networks:

- `blockchain_default` for Fabric and dashboard backend
- `qsnet` for watcher, signer, QS API, and dashboard backend

From the project root:

```bash
docker network create blockchain_default || true
docker network create qsnet || true
```

---

## 5. Start the Blockchain and QS API

Go to the blockchain folder:

```bash
cd blockchain
```

### 5.1 Generate Fabric channel artifacts

If `channel-artifacts/genesis.block` and `channel-artifacts/channel1.tx` already exist, you can skip this step.

If they do not exist, generate them:

```bash
mkdir -p channel-artifacts

docker run --rm \
  -v "$PWD/configtx:/etc/hyperledger/fabric" \
  -v "$PWD/channel-artifacts:/opt/channel-artifacts" \
  -e FABRIC_CFG_PATH=/etc/hyperledger/fabric \
  hyperledger/fabric-tools:2.5.14 \
  configtxgen \
  -profile OrdererGenesisEtcdraft \
  -channelID system-channel \
  -outputBlock /opt/channel-artifacts/genesis.block

docker run --rm \
  -v "$PWD/configtx:/etc/hyperledger/fabric" \
  -v "$PWD/channel-artifacts:/opt/channel-artifacts" \
  -e FABRIC_CFG_PATH=/etc/hyperledger/fabric \
  hyperledger/fabric-tools:2.5.14 \
  configtxgen \
  -profile MyAppChannelEtcdraft \
  -channelID channel1 \
  -outputCreateChannelTx /opt/channel-artifacts/channel1.tx
```

### 5.2 Start Fabric and QS API containers

```bash
docker compose up -d --build
```

Check containers:

```bash
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

Expected important containers:

```text
fabric-ca-server
orderer
orderer2
orderer3
endorsing_peer1
endorsing_peer2
commiting_peer
client
qs-api
```

Wait a few seconds for the orderer and peers to become ready:

```bash
docker logs orderer --tail 50
```

---

## 6. Create and Join the Fabric Channel

Run this from the `blockchain/` folder:

```bash
docker exec -i client bash <<'EOF'
set -e

export FABRIC_CFG_PATH=/etc/hyperledger/fabric
export CORE_PEER_LOCALMSPID=Org1MSP
export CORE_PEER_MSPCONFIGPATH=/opt/crypto/org1/admin/msp
export CORE_PEER_ADDRESS=endorsing_peer1:7051
export CORE_PEER_TLS_ENABLED=true
export CORE_PEER_TLS_ROOTCERT_FILE=/opt/crypto/org1/peer1/tls/tlscacerts/ca.pem
export ORDERER_CA=/opt/crypto/orderer/tls/tlscacerts/ca.pem

peer channel create \
  -o orderer:7050 \
  --ordererTLSHostnameOverride orderer \
  -c channel1 \
  -f /opt/channel-artifacts/channel1.tx \
  --outputBlock /opt/channel-artifacts/channel1.block \
  --tls \
  --cafile "$ORDERER_CA"

peer channel join -b /opt/channel-artifacts/channel1.block

CORE_PEER_ADDRESS=endorsing_peer2:8051 \
CORE_PEER_TLS_ROOTCERT_FILE=/opt/crypto/org1/peer2/tls/tlscacerts/ca.pem \
peer channel join -b /opt/channel-artifacts/channel1.block

CORE_PEER_ADDRESS=commiting_peer:9051 \
CORE_PEER_TLS_ROOTCERT_FILE=/opt/crypto/org1/peer3/tls/tlscacerts/ca.pem \
peer channel join -b /opt/channel-artifacts/channel1.block
EOF
```

If the channel already exists, Fabric may return an error saying the channel already exists. In that case, continue with the next step.

---

## 7. Install and Commit the Chaincode

Run this from the `blockchain/` folder:

```bash
docker exec -i client bash <<'EOF'
set -e

export FABRIC_CFG_PATH=/etc/hyperledger/fabric
export CORE_PEER_LOCALMSPID=Org1MSP
export CORE_PEER_MSPCONFIGPATH=/opt/crypto/org1/admin/msp
export CORE_PEER_ADDRESS=endorsing_peer1:7051
export CORE_PEER_TLS_ENABLED=true
export CORE_PEER_TLS_ROOTCERT_FILE=/opt/crypto/org1/peer1/tls/tlscacerts/ca.pem
export ORDERER_CA=/opt/crypto/orderer/tls/tlscacerts/ca.pem

peer lifecycle chaincode package /tmp/pqcCC_demo_final.tar.gz \
  --path /chaincode/pqcCC_demo \
  --lang golang \
  --label pqcCC_demo_final_1.0

peer lifecycle chaincode install /tmp/pqcCC_demo_final.tar.gz

PACKAGE_ID=$(peer lifecycle chaincode queryinstalled | sed -n 's/^Package ID: \(.*\), Label: pqcCC_demo_final_1.0/\1/p' | head -n 1)

echo "PACKAGE_ID=$PACKAGE_ID"

peer lifecycle chaincode approveformyorg \
  -o orderer:7050 \
  --ordererTLSHostnameOverride orderer \
  --channelID channel1 \
  --name pqcCC_demo_final \
  --version 1.0 \
  --package-id "$PACKAGE_ID" \
  --sequence 1 \
  --tls \
  --cafile "$ORDERER_CA"

peer lifecycle chaincode checkcommitreadiness \
  --channelID channel1 \
  --name pqcCC_demo_final \
  --version 1.0 \
  --sequence 1 \
  --output json

peer lifecycle chaincode commit \
  -o orderer:7050 \
  --ordererTLSHostnameOverride orderer \
  --channelID channel1 \
  --name pqcCC_demo_final \
  --version 1.0 \
  --sequence 1 \
  --tls \
  --cafile "$ORDERER_CA" \
  --peerAddresses endorsing_peer1:7051 \
  --tlsRootCertFiles /opt/crypto/org1/peer1/tls/tlscacerts/ca.pem

peer chaincode query \
  -C channel1 \
  -n pqcCC_demo_final \
  -c '{"Args":["GetAllEvents"]}'
EOF
```

Expected first query output may be:

```text
[]
```

That means the chaincode is installed and queryable, but no file events have been submitted yet.

---

## 8. Register the PQC Signer Public Key

The chaincode verifies every event using the registered signer public key. Register the demo signer key before submitting file events.

Run from the project root:

```bash
PUB=$(tr -d '\r\n' < qs-mtls-demo/pqc-signer/keys/dilithium_public.key)
SIGNER_KEY_JSON=$(jq -nc --arg id "pqc-signer-1" --arg pk "$PUB" '{id:$id,public_key_b64:$pk,active:true}')
CC_ARGS=$(jq -nc --arg sk "$SIGNER_KEY_JSON" '{Args:["RegisterSignerKey",$sk]}')
PAYLOAD_B64=$(printf "%s" "$CC_ARGS" | base64 -w0)

docker exec -i client bash <<EOF
set -e
export FABRIC_CFG_PATH=/etc/hyperledger/fabric
export CORE_PEER_LOCALMSPID=Org1MSP
export CORE_PEER_MSPCONFIGPATH=/opt/crypto/org1/admin/msp
export CORE_PEER_ADDRESS=endorsing_peer1:7051
export CORE_PEER_TLS_ENABLED=true
export CORE_PEER_TLS_ROOTCERT_FILE=/opt/crypto/org1/peer1/tls/tlscacerts/ca.pem
export ORDERER_CA=/opt/crypto/orderer/tls/tlscacerts/ca.pem

CC_ARGS=\$(echo "$PAYLOAD_B64" | base64 -d)

peer chaincode invoke \
  -o orderer:7050 \
  --ordererTLSHostnameOverride orderer \
  --tls \
  --cafile "\$ORDERER_CA" \
  -C channel1 \
  -n pqcCC_demo_final \
  --peerAddresses endorsing_peer1:7051 \
  --tlsRootCertFiles /opt/crypto/org1/peer1/tls/tlscacerts/ca.pem \
  -c "\$CC_ARGS"
EOF
```

Verify the signer key:

```bash
docker exec client bash -lc '
export FABRIC_CFG_PATH=/etc/hyperledger/fabric
export CORE_PEER_LOCALMSPID=Org1MSP
export CORE_PEER_MSPCONFIGPATH=/opt/crypto/org1/admin/msp
export CORE_PEER_ADDRESS=endorsing_peer1:7051
export CORE_PEER_TLS_ENABLED=true
export CORE_PEER_TLS_ROOTCERT_FILE=/opt/crypto/org1/peer1/tls/tlscacerts/ca.pem
peer chaincode query -C channel1 -n pqcCC_demo_final -c "{\"Args\":[\"ReadSignerKey\",\"pqc-signer-1\"]}"
'
```

---

## 9. Start the PQC Signer and Watcher

Go to the QS mTLS folder:

```bash
cd qs-mtls-demo
```

Create the watcher data folder if it does not exist:

```bash
mkdir -p watcher_data
```

Start the signer and watcher:

```bash
docker compose up -d --build
```

Check that both containers are running:

```bash
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}" | grep -E "watcher|pqc-signer"
```

Check watcher logs:

```bash
docker logs watcher --tail 100
```

Expected signs of success:

```text
Watching: /watch
Signer URL: https://pqc-signer:9443/sign
QS API logs path: /logs
[SEND] OK
```

---

## 10. Start the Dashboard

Go to the dashboard folder:

```bash
cd ../blockchain/dashboard
```

Start the dashboard backend and frontend:

```bash
docker compose up -d --build
```

Open the dashboard:

```text
http://localhost:3000
```

Default demo login:

```text
Username: admin
Password: admin123
```

Main API endpoint:

```text
http://localhost:9000/api/health
```

Frontend:

```text
http://localhost:3000
```

---

## 11. Quick End-to-End Test

From the project root, modify a watched file:

```bash
echo "# QuantumShield test $(date -u +%Y-%m-%dT%H:%M:%SZ)" >> qs-mtls-demo/watched_folder/etc/passwd
```

Watch the event flow:

```bash
docker logs watcher --tail 80
docker logs qs-api --tail 80
```

Query Fabric world state:

```bash
docker exec client bash -lc '
export FABRIC_CFG_PATH=/etc/hyperledger/fabric
export CORE_PEER_LOCALMSPID=Org1MSP
export CORE_PEER_MSPCONFIGPATH=/opt/crypto/org1/admin/msp
export CORE_PEER_ADDRESS=endorsing_peer1:7051
export CORE_PEER_TLS_ENABLED=true
export CORE_PEER_TLS_ROOTCERT_FILE=/opt/crypto/org1/peer1/tls/tlscacerts/ca.pem
peer chaincode query -C channel1 -n pqcCC_demo_final -c "{\"Args\":[\"GetAllEvents\"]}"
'
```

Then open the dashboard and check:

- Overview page
- Logs page
- World State page
- File History view
- Alerts page

---

## 12. Demo Scenario: Immutable Evidence

A strong demo scenario is:

1. A normal admin modifies `/watch/etc/passwd`.
2. The attacker adds a temporary unauthorized line.
3. The attacker removes the line to make the file look normal again.
4. QuantumShield still shows the full file history.
5. The attacker tries to submit a fake `DELETED` event with an invalid signature.
6. The system rejects the forged event with `signature mismatch`.
7. The dashboard shows the critical alert.

Example attacker line for demo only:

```text
ghost_admin:x:0:0:Unauthorized Admin:/root:/bin/bash
```

Main message:

```text
The attacker can change the file.
The attacker can restore the file.
But the attacker cannot rewrite the trusted history.
```

---

## 13. Optional: Forged Trace-Deletion Test

Use this only after the system is running and a real `file_id` exists.

Replace `PUT_REAL_FILE_ID_HERE` with the file ID shown in the dashboard for `/watch/etc/passwd`.

```bash
python3 - <<'PY'
import json
import uuid
import requests
from datetime import datetime, timezone

TARGET_FILE_ID = "PUT_REAL_FILE_ID_HERE"

payload = {
    "agent_id": "attacker-machine",
    "event_id": str(uuid.uuid4()),
    "event_type": "DELETED",
    "file_id": TARGET_FILE_ID,
    "file_name": "passwd",
    "path": "/watch/etc/passwd",
    "timestamp": datetime.now(timezone.utc).isoformat(),
    "old_sha256": "0" * 64,
    "reason": "attacker_attempted_to_delete_trace"
}

canonical_log = json.dumps(
    payload,
    ensure_ascii=False,
    sort_keys=True,
    separators=(",", ":")
)

envelope = {
    "Canonical_log": canonical_log,
    "Signature": "ZmFrZV9zaWduYXR1cmU=",
    "SignerID": "pqc-signer-1"
}

r = requests.post("http://localhost:8080/logs", json=envelope, timeout=10)

print("status:", r.status_code)
print("body:", r.text[:500])
PY
```

Expected result:

```text
status: 502
signature mismatch
```

Check QS API logs:

```bash
docker logs qs-api --tail 100 | grep -iE "signature mismatch|SECURITY|CRITICAL"
```

The dashboard backend watches container logs and should convert this into a critical alert.

---

## 14. Useful Health Checks

### QS API

```bash
curl http://localhost:8080/health
```

Expected:

```text
ok
```

### Dashboard API

```bash
curl http://localhost:9000/api/health
```

### Watcher logs

```bash
docker logs watcher --tail 100
```

### Signer logs

```bash
docker logs pqc-signer --tail 100
```

### Fabric logs

```bash
docker logs orderer --tail 100
docker logs endorsing_peer1 --tail 100
```

### Dashboard backend alerts

```bash
docker logs fim-api --tail 100 | grep -iE "ALERT|CRITICAL|signature|qs-api"
```

---

## 15. Common Problems

### Problem: `network qsnet declared as external, but could not be found`

Create the network:

```bash
docker network create qsnet || true
```

### Problem: `network blockchain_default declared as external, but could not be found`

Create the network:

```bash
docker network create blockchain_default || true
```

### Problem: QS API is unreachable from watcher

Check:

```bash
docker exec watcher python - <<'PY'
import requests
for url in ["http://host.docker.internal:8080/health", "http://qs-api:8080/health"]:
    try:
        r = requests.get(url, timeout=5)
        print(url, r.status_code, r.text[:100])
    except Exception as e:
        print(url, type(e).__name__, e)
PY
```

Use the route that works in your Docker setup.

### Problem: dashboard does not show alerts

Check that `fim-api` is watching `qs-api` logs:

```bash
docker logs fim-api --tail 200 | grep -iE "LogStream|Connected to qs-api|ALERT|CRITICAL"
```

If needed, restart the dashboard backend:

```bash
cd blockchain/dashboard
docker compose restart fim-api
```

### Problem: `signature mismatch`

This usually means the event was modified after signing or the signature is fake. For security tests, this is expected. For normal watcher events, check that the signer public key registered on-chain matches the private key used by `pqc-signer`.

### Problem: `file asset ... does not exist`

An update or delete event was submitted for a `file_id` that does not currently exist in Fabric world state. Create the file first, then modify or delete it.

---

## 16. Stop the System

Stop dashboard:

```bash
cd blockchain/dashboard
docker compose down
```

Stop watcher and signer:

```bash
cd ../../qs-mtls-demo
docker compose down
```

Stop blockchain:

```bash
cd ../blockchain
docker compose down
```

To remove containers and volumes completely:

```bash
docker compose down -v
```

Use volume deletion carefully because it removes Fabric ledger state.

---

## 17. Security Notes

This repository may contain demo keys, certificates, and local test data. For GitHub/public release:

- Do not use demo private keys in production.
- Regenerate mTLS certificates before any real deployment.
- Regenerate ML-DSA keys before any real deployment.
- Do not commit real private keys, production certificates, or sensitive ledger data.
- Use `.gitignore` for generated files such as local databases, private keys, and Docker volumes.

Recommended `.gitignore` entries:

```gitignore
*.db
*.sqlite
*.key
*.csr
*.srl
watcher_data/
security_evidence/
channel-artifacts/*.block
channel-artifacts/*.tx
```

Keep only safe demo material in the public repository.

---

## 18. Project Summary

QuantumShield demonstrates a post-quantum, blockchain-backed audit trail for file integrity monitoring.

It does not promise that attackers will never touch a file. It proves that once a file is touched, the evidence can be preserved, verified, and reconstructed.

Final demo message:

```text
QuantumShield is not a wall.
It is a witness that cannot be silenced.
```
