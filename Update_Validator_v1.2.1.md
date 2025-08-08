# UPGRADE VALIDATOR TO v1.2.1
---
Go to: https://0g-labs.notion.site/0G-Validator-Node-Upgrade-Guide-v1-2-1-Release-248d6515e14380d6bcb4f750941deb16
Download galileo-v1.2.1.zip and transfer to VPS
---
## Upgrade Steps
### Step 1: Extract the New Release
```bash
unzip -o galileo-v1.2.1.zip
```
### Step 2: Stop existing Services
```bash
sudo systemctl stop geth 0gchaind
```
### Step 3: Replace the galileo Folder
```bash
mv galileo galileo-backup-$(date +%Y%m%d-%H%M%S)
mv galileo-v1.2.1 galileo
```
### Step 4: Replace geth and 0gchaind Binaries
```bash
sudo chmod +x ~/galileo/bin/geth
sudo chmod +x ~/galileo/bin/0gchaind
sudo cp -f ~/galileo/bin/geth /usr/local/bin/geth
sudo cp -f ~/galileo/bin/0gchaind /usr/local/bin/0gchaind
```
### Step 5: Backup Your Data
```bash
# Create backup directory with timestamp
BACKUP_DIR="backup-v121-$(date +%Y%m%d-%H%M%S)"
mkdir -p $BACKUP_DIR

# Define data paths
export GETH_DATADIR=$HOME/.0gchaind/galileo/0g-home/geth-home
export CL_HOME=$HOME/.0gchaind/galileo/0g-home/0gchaind-home

# Backup Execution Layer data
rsync -a $GETH_DATADIR/ $BACKUP_DIR/geth-backup/

# Backup Consensus Layer data
rsync -a $CL_HOME/ $BACKUP_DIR/cl-backup/
```
### Step 6: Create or Update geth.service
```bash
sudo rm -f /etc/systemd/system/geth.service /etc/systemd/system/0gchaind.service
```
```bash
USER_NAME=$(whoami)
HOME_DIR=$(eval echo ~$USER_NAME)
PUBLIC_IP=$(curl -s ifconfig.me)

sudo tee /etc/systemd/system/geth.service > /dev/null <<EOF
[Unit]
Description=0g Geth Node Service (Galileo v1.2.1)
After=network-online.target

[Service]
User=$USER_NAME
WorkingDirectory=$HOME_DIR/.0gchaind/galileo
ExecStart=/usr/local/bin/geth \\
  --config $HOME_DIR/.0gchaind/galileo/geth-config.toml \\
  --nat extip:$PUBLIC_IP \\
  --bootnodes enode://de7b86d8ac452b1413983049c20eafa2ea0851a3219c2cc12649b971c1677bd83fe24c5331e078471e52a94d95e8cde84cb9d866574fec957124e57ac6056699@8.218.88.60:30303 \\
  --datadir $HOME_DIR/.0gchaind/galileo/0g-home/geth-home \\
  --networkid 16601 \\
  --port 26303 \\
  --http.port 26545 \\
  --ws.port 26546 \\
  --authrpc.port 26551
Restart=always
RestartSec=3
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF
```
```bash
sudo systemctl daemon-reload
sudo systemctl start geth
```
### Step 7: Run Consensus Layer Rollback
```bash
# Replace all "beacon-kit" with "chaincfg" in app.toml
sed -i 's/beacon-kit/chaincfg/g' $CL_HOME/config/app.toml

# Run rollback script
cd ~/galileo
sh ./rollback_cl.sh $CL_HOME 127.0.0.1:26545
# 26545 change to rpc port of your validator rpc ( default 8545 )
```
```bash
sudo systemctl stop geth
```
### Step 8: Reset Validator State
```bash
# Backup the current validator state
cp $CL_HOME/data/priv_validator_state.json \
   $CL_HOME/data/priv_validator_state.json.backup-$(date +%Y%m%d-%H%M%S)

# Remove the old state file
rm -f $CL_HOME/data/priv_validator_state.json

# Create a fresh state file
cat > $CL_HOME/data/priv_validator_state.json <<EOF
{
  "height": "0",
  "round": 0,
  "step": 0
}
EOF

# Set ownership and permissions
chown $USER:$USER $CL_HOME/data/priv_validator_state.json
chmod 644 $CL_HOME/data/priv_validator_state.json
```
### Step 9: Create or Update 0gchaind.service
```bash
sudo tee /etc/systemd/system/0gchaind.service > /dev/null <<EOF
[Unit]
Description=0gchaind Node Service (Galileo v1.2.1)
After=network-online.target

[Service]
User=$USER_NAME
Environment=CHAIN_SPEC=devnet
WorkingDirectory=$HOME_DIR/.0gchaind/galileo
ExecStart=/usr/local/bin/0gchaind start \\
  --rpc.laddr tcp://0.0.0.0:26657 \\
  --chaincfg.chain-spec devnet \\
  --chaincfg.kzg.trusted-setup-path=$HOME_DIR/.0gchaind/galileo/kzg-trusted-setup.json \\
  --chaincfg.engine.jwt-secret-path=$HOME_DIR/.0gchaind/galileo/jwt-secret.hex \\
  --chaincfg.kzg.implementation=crate-crypto/go-kzg-4844 \\
  --chaincfg.block-store-service.enabled \\
  --chaincfg.node-api.enabled \\
  --chaincfg.node-api.logging \\
  --chaincfg.node-api.address 0.0.0.0:3500 \\
  --pruning=nothing \\
  --home $HOME_DIR/.0gchaind/galileo/0g-home/0gchaind-home \\
  --p2p.seeds=85a9b9a1b7fa0969704db2bc37f7c100855a75d9@8.218.88.60:26656 \\
  --p2p.external_address=$PUBLIC_IP:26656
Restart=always
RestartSec=3
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF

```
### Step 10: Start Services
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now geth 0gchaind

```
### Step 11: Verify Services
```bash
# Kiểm tra trạng thái
sudo systemctl status geth --no-pager -l
sudo systemctl status 0gchaind --no-pager -l
```
