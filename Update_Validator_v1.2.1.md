# UPGRADE VALIDATOR TO v1.2.1
---
Go to: https://0g-labs.notion.site/0G-Validator-Node-Upgrade-Guide-v1-2-1-Release-248d6515e14380d6bcb4f750941deb16
Download galileo-v1.2.1.zip and transfer to VPS
---
## Upgrade Steps
### Step 1: Extract New Release
```bash
unzip -o galileo-v1.2.1.zip
```
### Step 2: Stop Services
```bash
sudo systemctl stop geth 0gchaind
```
### replace geth and 0gchaind binary
```bash
sudo chmod +x ~/galileo/bin/geth
sudo chmod +x ~/galileo/bin/0gchaind
sudo cp -f ~/galileo/bin/geth /usr/local/bin/geth
sudo cp -f ~/galileo/bin/0gchaind /usr/local/bin/0gchaind
```
### Step 3: Backup Your Data
```bash
# Create backup directory with timestamp
BACKUP_DIR="backup-v120-$(date +%Y%m%d-%H%M%S)"
mkdir -p $BACKUP_DIR
export your_geth_datadir=~/.0gchaind/galileo/0g-home/geth-home
export your_cl_home=~/.0gchaind/galileo/0g-home/0gchaind-home

# Backup execution layer data(geth-home)
rsync -a $your_geth_datadir/ $BACKUP_DIR/geth-backup/

# Backup consensus layer data (0gchaind-home)
rsync -a $your_cl_home/ $BACKUP_DIR/cl-backup/
```
```bash
mv galileo galileo-backup-$(date +%Y%m%d-%H%M%S)
mv galileo-v1.2.1 galileo
```
### Step 4: Run Consensus Layer Rollback
```bash
# Replace all "beacon-kit" with "chaincfg" in app.toml
sed -i 's/beacon-kit/chaincfg/g' $your_cl_home/config/app.toml

# Then retry the rollback script
cd ~/galileo
sh ./rollback_cl.sh $your_cl_home 127.0.0.1:8545
```
### Step 5: Remove Validator State File
```bash
# Backup the current validator state (optional but recommended)
cp $your_cl_home/data/priv_validator_state.json \
   $your_cl_home/data/priv_validator_state.json.backup-$(date +%Y%m%d-%H%M%S)

# Remove the state file (will be regenerated)
rm $your_cl_home/data/priv_validator_state.json
```
```bash
cat > $HOME/.0gchaind/galileo/0g-home/0gchaind-home/data/priv_validator_state.json <<EOF
{
  "height": "0",
  "round": 0,
  "step": 0
}
EOF
```
```bash
chown $USER:$USER $HOME/.0gchaind/galileo/0g-home/0gchaind-home/data/priv_validator_state.json
```
### geth.service
### 0gchaind.service
```bash
sudo rm -f /etc/systemd/system/geth.service /etc/systemd/system/0gchaind.service
```
```bash
USER_NAME=$(whoami)
HOME_DIR=$(eval echo ~$USER_NAME)
PUBLIC_IP=$(curl -s ifconfig.me)

# Tạo geth.service
cat <<EOF | sudo tee /etc/systemd/system/geth.service
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

# Tạo 0gchaind.service
cat <<EOF | sudo tee /etc/systemd/system/0gchaind.service
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
### Start service
```bash
# Reload & khởi động dịch vụ
sudo systemctl daemon-reload
sudo systemctl enable geth 0gchaind
sudo systemctl start geth 0gchaind
```
### Check logs
```bash
# Kiểm tra trạng thái
sudo systemctl status geth --no-pager -l
sudo systemctl status 0gchaind --no-pager -l
```
