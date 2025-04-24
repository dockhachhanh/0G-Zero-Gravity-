Hướng dẫn dựa trên hướng dẫn của bạn Josephtran
link gốc: https://service.josephtran.xyz/testnet/zero-gravity-0g/0g-storage-node/installation
---
# HƯỚNG DẪN CÀI ĐẶT 0G STORAGE NODE - TESTNET V3 GALILEO
## 1. Install dependencies for building from source
```bash
sudo apt-get update
sudo apt-get install clang cmake build-essential openssl pkg-config libssl-dev
```
## 2. install go version 1.24.2
```bash
cd $HOME && \
ver="1.24.2" && \
wget "https://golang.org/dl/go$ver.linux-amd64.tar.gz" && \
sudo rm -rf /usr/local/go && \
sudo tar -C /usr/local -xzf "go$ver.linux-amd64.tar.gz" && \
rm "go$ver.linux-amd64.tar.gz" && \
echo "export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin" >> ~/.bash_profile && \
source ~/.bash_profile && \
go version
```
## 3. install rustup
```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
. "$HOME/.cargo/env"
```
## 4. Download and install binary
```bash
git clone -b v1.0.0 https://github.com/0glabs/0g-storage-node.git
cd 0g-storage-node
cargo build --release
```
## 5. Download config file:
```bash
curl -o $HOME/0g-storage-node/run/config-testnet-turbo.toml https://cdn.bangcode.id/0g/v3_config.toml
```
