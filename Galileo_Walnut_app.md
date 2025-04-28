# HƯỚNG DẪN CHI TIẾT
# TRIỂN KHAI WALNUT APP TRÊN 0G GALILEO TESTNET
---
## Mục tiêu
- Triển khai hợp đồng Walnut.sol (mô phỏng trò chơi multiplayer) trên 0G Galileo testnet.
- Xây dựng CLI để tương tác với hợp đồng (thực hiện các hành động hit, shake, reset, look).
- Xác minh hợp đồng trên chainscan-galileo.0g.ai để công khai mã nguồn và ABI.
- Đảm bảo CLI chạy kịch bản tương tự hướng dẫn gốc (Round 1: Alice, Round 2: Bob, kiểm tra quyền truy cập).
## Yêu cầu
- Hệ điều hành: Ubuntu 22.04 LTS (khuyến nghị), Mac, hoặc Windows (với Terminal/Power работаетShell).
## Công cụ:
- Node.js và npm: Phiên bản 18.x hoặc 20.x.
- Hardhat: Để biên dịch và triển khai hợp đồng.
- Bun: Để chạy CLI (hoặc Node.js nếu không dùng Bun).
- viem: Thư viện thay thế seismic-viem để tương tác với hợp đồng.
- Ví Ethereum: 2 ví (Alice và Bob) với ít nhất 0.1 token testnet mỗi ví từ https://faucet.0g.ai/ (yêu cầu tài khoản X có >10 follower).
- Kết nối internet: Để tải gói, nạp token testnet, và tương tác với Galileo.
- Kiến thức cơ bản: Sử dụng terminal, chỉnh sửa file văn bản.
## Cấu trúc dự án
- Dự án sẽ có cấu trúc sau:
```
walnut-galileo/
├── contracts/              # Hợp đồng Solidity
│   └── Walnut.sol
├── scripts/                # Script triển khai
│   └── deploy.js
├── cli/                    # CLI để tương tác
│   ├── src/
│   │   ├── app.ts
│   │   └── index.ts
│   ├── lib/
│   │   ├── constants.ts
│   │   └── utils.ts
│   ├── artifacts/
│   │   └── contracts/
│   │       └── Walnut.sol/
│   │           └── Walnut.json
│   ├── .env
│   ├── .gitignore
│   ├── package.json
│   ├── package-lock.json
│   └── tsconfig.json
├── artifacts/              # File biên dịch hợp đồng
├── .env
├── .gitignore
├── hardhat.config.js
├── package.json
├── package-lock.json
```
---
## Bước 1: Chuẩn bị môi trường
### Cài đặt Node.js và npm:
Kiểm tra xem *Node.js*đã cài chưa:
```bash
node -v
npm -v
```
Nếu chưa có hoặc phiên bản < *18.x*, cài đặt:
```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.2/install.sh | bash
source ~/.bashrc
nvm install 20
node -v  # Nên hiển thị v20.x.x
npm -v  # Nên hiển thị 10.x.x
```
### Cài đặt Bun:
*Bun* dùng để chạy CLI (nhanh hơn *Node.js*):
```bash
curl -fsSL https://bun.sh/install | bash
source ~/.bashrc
bun -v  # Nên hiển thị 1.x.x
```
### Cập nhật hệ thống (Ubuntu):
Đảm bảo hệ thống có các gói cơ bản:
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install curl git wget jq build-essential -y
```
### Chuẩn bị ví Ethereum:
- Tạo 2 ví *(Alice và Bob)* qua MetaMask hoặc công cụ tương tự.
- Lấy private key của mỗi ví (không cần 0x).
#### Nạp token testnet:
-- Truy cập *https://faucet.0g.ai/*.
-- Đăng nhập bằng tài khoản X (*>10 follower*).
-- Nhập địa ငhỉ ví của *Alice* và *Bob*, yêu cầu ít nhất 0.1 token testnet cho mỗi ví.
-- Kiểm tra số dư trên https://chainscan-galileo.0g.ai/ bằng địa chỉ ví.

## Tạo thư mục dự án:
```bash
mkdir ~/walnut-galileo
cd ~/walnut-galileo
npm init -y
```
---
## Bước 2: Thiết lập dự án Hardhat
### Cài đặt Hardhat và phụ thuộc:
```bash
npm install --save-dev hardhat @nomicfoundation/hardhat-toolbox @nomicfoundation/hardhat-verify dotenv
npm install viem
```
### Khởi tạo dự án Hardhat:
```bash
npx hardhat
```
Chọn Create a JavaScript project.
Nhấn Enter để đồng ý cài đặt phụ thuộc.
- Kiểm tra Hardhat:
```bash
npx hardhat --version  # Nên hiển thị ≥ 2.17.0
```
### Cấu hình *.env*:
Tạo file *.env*:
```bash
nano .env
```
Thêm nội dung:
```
RPC_URL=https://evmrpc-testnet.0g.ai/
CHAIN_ID=80087
ALICE_PRIVATE_KEY=<alice-private-key>
BOB_PRIVATE_KEY=<bob-private-key>
CHAINSCAN_API_KEY=abc
```
- Thay *<alice-private-key>* và *<bob-private-key>* bằng khóa riêng của ví.
- Giữ CHAINSCAN_API_KEY=abc trừ khi 0G Network yêu cầu key khác (liên hệ qua Discord nếu cần).
- Lưu và thoát.
### Cấu hình *.gitignore*:
Tạo file *.gitignore*:
```bash
nano .gitignore
```
Thêm nội dung:
```
node_modules
.env
artifacts
cache
```
Lưu và thoát.
### Cấu hình Hardhat:
Mở *hardhat.config.js*:
```bash
nano hardhat.config.js
```
Thay bằng nội dung:
```javascript
require("@nomicfoundation/hardhat-toolbox");
require("@nomicfoundation/hardhat-verify");
require("dotenv").config();

module.exports = {
  solidity: {
    compilers: [
      {
        version: "0.8.20",
        settings: {
          optimizer: {
            enabled: true,
            runs: 200
          },
          evmVersion: "istanbul"
        }
      }
    ]
  },
  networks: {
    galileo: {
      url: process.env.RPC_URL || "https://evmrpc-testnet.0g.ai",
      accounts: [
        process.env.ALICE_PRIVATE_KEY || "",
        process.env.BOB_PRIVATE_KEY || ""
      ],
      chainId: 80087,
      gas: 5000000,
      gasPrice: 1000000000
    }
  },
  etherscan: {
    apiKey: {
      galileo: process.env.CHAINSCAN_API_KEY || "abc"
    },
    customChains: [
      {
        network: "galileo",
        chainId: 80087,
        urls: {
          apiURL: "https://chainscan-galileo.0g.ai/api",
          browserURL: "https://chainscan-galileo.0g.ai"
        }
      }
    ]
  },
  sourcify: {
    enabled: false
  }
};
```
Lưu và thoát.
### Kiểm tra RPC:
Đảm bảo RPC hoạt động:
```bash
curl -X POST -H "Content-Type: application/json" --data '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}' https://evmrpc-testnet.0g.ai/
```
Nếu trả về số block (ví dụ: ``` {"jsonrpc":"2.0","result":"0x1234","id":1}) ```, RPC ổn.


