# Upgrade-Aztec-SequencerNode
## part1: Install Required Tools

### Install Foundry
```bash
curl -L https://foundry.paradigm.xyz | bash
```

**Reload your shell:**
```bash
source ~/.bashrc
```

**Install Foundry tools:**
```bash
foundryup
```

**Verify installation:**
```bash
cast --version
```

### Install Aztec CLI

```bash
bash -i <(curl -s https://install.aztec.network)
```

**Add Aztec CLI to PATH:**
```bash
echo 'export PATH="$HOME/.aztec/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

**Install specific Aztec CLI version:**
```bash
aztec-up 2.1.2
```

**Verify installation:**
```bash
aztec --version
```

You should see the Aztec CLI version (2.1.2).

## Part 2: Create Validator Keys

Now that you have Aztec CLI installed, let's create your validator keys.

### Generate Keystore with Aztec CLI

```bash
aztec validator-keys new \
  --fee-recipient 0x0000000000000000000000000000000000000000000000000000000000000000
```
save it somewhere. if you have done this before, use `cat ~/.aztec/keystore/key1.json` to see your keys.
fund your attester address with min 0.2 sepolia eth

## Part 3: Register Your Validator on the Network

```bash
aztec \
  add-l1-validator \
  --l1-rpc-urls $ETH_RPC \
  --network testnet \
  --private-key $PRIVATE_KEY_OF_OLD_SEQUENCER \
  --attester $ETH_ATTESTER_ADDRESS \
  --withdrawer $ANY_ETH_ADDRESS \
  --bls-secret-key $BLS_ATTESTER_PRIV_KEY \
  --rollup 0xebd99ff0ff6677205509ae73f93d0ca52ac85d67
```
you can use a public rpc like `https://0xrpc.io/sep` . use the attester and bls key you got before. a pv key to pay gas and a withdrawer address to withdraw your stake.
check if you have joined the validator queue in https://dashtec.xyz/queue by searching attester address.

## Part 4: Set Up Node Directory Structure

### Delete the existing configuration
```bash
cd /root/aztec && docker compose down 
```
```bash
cd && rm -rf aztec
```
### Create Node Directory

```bash
mkdir -p /root/aztec/keys
mkdir -p /root/aztec/data
cd /root/aztec
```

**Directory structure:**
- `/root/aztec/keys/` - Contains keystore files
- `/root/aztec/data/` - Contains node data
- `/root/aztec/` - Contains docker-compose.yml and .env files

### Copy and Update Keystore
```bash
# Read values from your original keystore
ETH_KEY=$(jq -r '.validators[0].attester.eth' ~/.aztec/keystore/key1.json)
BLS_KEY=$(jq -r '.validators[0].attester.bls' ~/.aztec/keystore/key1.json)
FEE_RECIPIENT=$(jq -r '.validators[0].feeRecipient // "0x0000000000000000000000000000000000000000000000000000000000000000"' ~/.aztec/keystore/key1.json)

# Create the node keystore with coinbase field
cat > /root/aztec/keys/keystore.json <<EOF
{
  "schemaVersion": 1,
  "validators": [
    {
      "attester": {
        "eth": "$ETH_KEY",
        "bls": "$BLS_KEY"
      },
      "coinbase": "<YOUR_COINBASE_ADDRESS>",
      "feeRecipient": "$FEE_RECIPIENT"
    }
  ]
}
EOF
```
**Replace `<YOUR_COINBASE_ADDRESS>`** with the Ethereum address that should receive L1 block rewards (usually your withdrawer address or the account that funded registration).
```bash
chmod 600 /root/aztec/keys/keystore.json
```

### check the file
```bash
cat /root/aztec/keys/keystore.json | jq .
```

**Expected format:**
```json
{
  "schemaVersion": 1,
  "validators": [
    {
      "attester": {
        "eth": "0x...",
        "bls": "0x..."
      },
      "coinbase": "0x...",
      "feeRecipient": "0x..."
    }
  ]
}
```

## Part 5: Configure Docker Compose

### Create .env File

```bash
cd /root/aztec
nano .env
```

**Paste This and Use your Ip and Rpc url**
```bash
ETHEREUM_RPC_URL=<execution-rpc-url>
CONSENSUS_BEACON_URL=<consensus-rpc-url>
GOVERNANCE_PROPOSER_PAYLOAD_ADDRESS=0xDCd9DdeAbEF70108cE02576df1eB333c4244C666
P2P_IP=<YOUR_PUBLIC_IP_ADDRESS>
P2P_PORT=40400
AZTEC_PORT=8080
LOG_LEVEL=info
```
save and exit

### Create docker-compose.yml

```bash
cd /root/aztec
nano docker-compose.yml
```

**Add this configuration without changing anything**
```yaml
services:
  aztec-node:
    container_name: aztec-sequencer
    image: aztecprotocol/aztec:2.1.2
    restart: unless-stopped
    network_mode: host
    environment:
      GOVERNANCE_PROPOSER_PAYLOAD_ADDRESS: ${GOVERNANCE_PROPOSER_PAYLOAD_ADDRESS}
      ETHEREUM_HOSTS: ${ETHEREUM_RPC_URL}
      L1_CONSENSUS_HOST_URLS: ${CONSENSUS_BEACON_URL}
      DATA_DIRECTORY: /var/lib/data
      KEY_STORE_DIRECTORY: /var/lib/keystore
      P2P_IP: ${P2P_IP}
      P2P_PORT: ${P2P_PORT:-40400}
      AZTEC_PORT: ${AZTEC_PORT:-8080}
      LOG_LEVEL: ${LOG_LEVEL:-info}
    entrypoint: >
      sh -c 'node --no-warnings /usr/src/yarn-project/aztec/dest/bin/index.js start --network testnet --node --archiver --sequencer --snapshots-urls https://s3.us-east-1.amazonaws.com/aztec-testnet-snapshots'
    ports:
      - 40400:40400/tcp
      - 40400:40400/udp
      - 8080:8080
    volumes:
      - /root/aztec/data:/var/lib/data
      - /root/aztec/keys:/var/lib/keystore
```
## Part 6: Start the Node

```bash
cd /root/aztec
docker compose up -d
```

### Check the logs

```bash
docker compose logs -fn 200
```
