# Hướng Dẫn Cài Đặt Node 0GChain Galileo v1.0.1
---
##✅ Yêu cầu hệ thống:

| Component      | Mainnet                     | Testnet                    |
|----------------|-----------------------------|----------------------------|
| **Memory**     | 64 GB                       | 64 GB                      |
| **CPU**        | 8 cores                     | 8 cores                    |
| **Disk**       | 1 TB NVME SSD               | 4 TB NVME SSD              |
| **Bandwidth**  | 100 MBps Download / Upload  | 100 MBps Download / Upload |

---
## Bước 1: Cài đặt gói phụ thuộc
```bash
sudo apt update
sudo apt install curl git make jq build-essential gcc unzip wget lz4 aria2 -y
```
## Bước 2: Tải và giải nén gói phần mềm
```bash
wget https://github.com/0glabs/0gchain-ng/releases/download/v1.0.1/galileo-v1.0.1.tar.gz
tar -xzvf galileo-v1.0.1.tar.gz -C $HOME
```
## Bước 3: Tạo thư mục .0gchain, cấp quyền
```bash
mkdir -p .0gchain
cd ~/galileo
cp -r 0g-home $HOME/.0gchain
sudo chmod 777 ./bin/geth
sudo chmod 777 ./bin/0gchaind
```
## Bước 4: Khởi tạo geth và 0gchaind
```bash
./bin/geth init --datadir $HOME/.0gchain/0g-home/geth-home ./genesis.json
```
## Bước 5: Khởi tạo geth và 0gchaind
```bash
./bin/0gchaind init DocKhachHanh --home $HOME/.0gchain/tmp
```
- thay tên DocKhachHanh bằng tên mà bạn muốn
## Bước 6: Sao chép file cấu hình
```
cp $HOME/.0gchain/tmp/data/priv_validator_state.json $HOME/.0gchain/0g-home/0gchaind-home/data/
cp $HOME/.0gchain/tmp/config/node_key.json $HOME/.0gchain/0g-home/0gchaind-home/config/
cp $HOME/.0gchain/tmp/config/priv_validator_key.json $HOME/.0gchain/0g-home/0gchaind-home/config/
```
## Bước 7: Tạo systemd service cho 0gchaind
```bash
sudo tee /etc/systemd/system/0gd.service > /dev/null <<EOF
[Unit]
Description=0gchaind Node Service
After=network-online.target

[Service]
User=$USER
WorkingDirectory=/home/$USER/galileo
ExecStart=/bin/bash -c 'cd ~/galileo && CHAIN_SPEC=devnet ./bin/0gchaind start \
    --rpc.laddr tcp://0.0.0.0:26657 \
    --beacon-kit.kzg.trusted-setup-path=kzg-trusted-setup.json \
    --beacon-kit.engine.jwt-secret-path=jwt-secret.hex \
    --beacon-kit.kzg.implementation=crate-crypto/go-kzg-4844 \
    --beacon-kit.block-store-service.enabled \
    --beacon-kit.node-api.enabled \
    --beacon-kit.node-api.logging \
    --beacon-kit.node-api.address 0.0.0.0:3500 \
    --pruning=nothing \
    --home $HOME/.0gchain/0g-home/0gchaind-home \
    --p2p.external_address $(wget -qO- eth0.me):26656 \
    --p2p.seeds b30fb241f3c5aee0839c0ea55bd7ca18e5c855c1@8.218.94.246:26656'
Restart=always
RestartSec=3
LimitNOFILE=4096

[Install]
WantedBy=multi-user.target
EOF
```
## Bước 8: Tạo systemd service cho geth
```bash
sudo tee /etc/systemd/system/geth.service > /dev/null <<EOF
[Unit]
Description=0g Geth Node Service
After=network-online.target

[Service]
User=$USER
WorkingDirectory=/home/$USER/galileo
ExecStart=/bin/bash -c 'cd ~/galileo && ./bin/geth --config geth-config.toml --datadir $HOME/.0gchain/0g-home/geth-home --networkid 80087'
Restart=always
RestartSec=3
LimitNOFILE=4096

[Install]
WantedBy=multi-user.target
EOF
```
## Bước 9: Bật và khởi động dịch vụ
```bash
sudo systemctl daemon-reexec
sudo systemctl daemon-reload
sudo systemctl enable 0gd
sudo systemctl enable geth
sudo systemctl start 0gd
sudo systemctl start geth
```
## Bước 10: Kiểm tra trạng thái node
```bash
sudo systemctl status 0gd
sudo systemctl status geth
```
## Check log
```bash
sudo journalctl -fu 0gd geth
```
---
Chúc các bạn thành công.
Link tham khảo: https://service.josephtran.xyz/testnet/zero-gravity-0g/validator-node/installation
