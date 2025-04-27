# Hướng dẫn chi tiết: Triển khai và xác minh token ERC-20 trên mạng Galileo testnet
## Mục tiêu
- Triển khai một hợp đồng token ERC-20 (MyToken) trên Galileo testnet của 0G Network (chain ID 80087).
- Xác minh hợp đồng trên chainscan-galileo.0g.ai để công khai mã nguồn và lấy ABI.

## Điều kiện tiên quyết
- Node.js và npm: Cài đặt phiên bản mới nhất (ví dụ: Node.js 18.x hoặc 20.x).
- Ví Ethereum: Có private key và đủ token testnet (0G testnet token) để trả gas.
- faucet: faucet.0g.ai ( X có hơn 10 follower )
- RPC URL: https://evmrpc-testnet.0g.ai/.
- Chainscan: chainscan-galileo.0g.ai để kiểm tra và xác minh hợp đồng.
- Công cụ: Terminal (Linux/Mac) hoặc Command Prompt/PowerShell (Windows).
- Kiến thức cơ bản: Hiểu về Solidity, Hardhat, và triển khai hợp đồng.
---
## Bước 1: Thiết lập dự án Hardhat
Tạo thư mục dự án:
```bash
mkdir my-token
cd my-token
npm init -y
```
Cài đặt Hardhat và các phụ thuộc:
```bash
npm install --save-dev hardhat @nomicfoundation/hardhat-toolbox @nomicfoundation/hardhat-verify dotenv
npm install @openzeppelin/contracts
```
Khởi tạo dự án Hardhat:
```bash
npx hardhat
```
- Chọn Create a JavaScript project.
Đồng ý cài đặt các phụ thuộc đề xuất.
Tạo file .env:
Trong thư mục gốc, tạo file .env:
```
RPC_URL=https://evmrpc-testnet.0g.ai/
PRIVATE_KEY=<YOUR_PRIVATE_KEY> 
CHAINSCAN_API_KEY=abc
```
- Thay <YOUR_PRIVATE_KEY> bằng private key của ví (bỏ 0x, ví dụ: private key của ví 0xE7dE245F746A43fE50Bf3dBE668D564372E82C5b).
- CHAINSCAN_API_KEY để "abc" trừ khi chainscan yêu cầu key (liên hệ 0G Network qua Discord nếu cần).
  
### Cấu hình Hardhat:
Mở file ```hardhat.config.js``` và thay bằng:
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
      url: process.env.RPC_URL || "https://rpc-testnet.0g.ai",
      accounts: process.env.PRIVATE_KEY ? [process.env.PRIVATE_KEY] : [],
      chainId: 80087,
      gas: 5000000, // Tránh lỗi gas
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
          apiURL: "https://chainscan-galileo.0g.ai/api", // Có thể cần thay đổi
          browserURL: "https://chainscan-galileo.0g.ai"
        }
      }
    ]
  },
  sourcify: {
    enabled: false // Tắt Sourcify vì không hỗ trợ chain ID 80087
  }
};
```
Cài đặt phụ thuộc:
- Đảm bảo tất cả phụ thuộc được cài đặt:
```bash
npm install
```
- Kiểm tra phiên bản Hardhat:
```bash
npx hardhat --version
```
-- Đảm bảo phiên bản ≥ 2.17.0. Nếu không, cập nhật:
```bash
npm install --save-dev hardhat@latest
```
---
## Bước 2: Viết hợp đồng MyToken
Tạo file hợp đồng:
Trong thư mục contracts/, tạo file MyToken.sol:
```bash
nano contracts/MyToken.sol
```
nhập nội dung sau:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";

contract MyToken is ERC20 {
    constructor(uint256 initialSupply) ERC20("My Token", "MTK") {
        _mint(msg.sender, initialSupply);
    }
}
```
Giải thích:
- Hợp đồng kế thừa ERC20 từ OpenZeppelin.
- Constructor nhận initialSupply (số token ban đầu, 18 decimals).
- Mint token cho msg.sender (ví triển khai).
- Tên: "My Token", ký hiệu: "MTK".
- Biên dịch hợp đồng:
```bash
npx hardhat compile
```

Đảm bảo không có lỗi. Artifact sẽ được lưu trong artifacts/.
Nếu gặp lỗi HH606: Solidity version mismatch (như với Lock.sol dùng 0.8.28)
```bash
nano contracts/Lock.sol
```
Xác nhận ```pragma solidity ^0.8.20 ```; nếu khác sửa cho đúng
- Biên dịch lại hợp đồng.
- kết quả:
```
Compiled 7 Solidity files successfully (evm target: istanbul).
```
---
## Bước 3: Viết và chạy script triển khai
Tạo script triển khai:
```bash
mkdir scripts
```
Tạo file ```bash nano deploy.js ``` với nội dung:
```javascript
const hre = require("hardhat");

async function main() {
  // Lấy signer (ví triển khai)
  const [deployer] = await hre.ethers.getSigners();
  console.log("Deploying from:", deployer.address);

  // Định dạng initialSupply
  const initialSupply = hre.ethers.parseUnits("1000000", 18); // 1 triệu token, 18 decimals

  // Tạo contract factory
  const MyToken = await hre.ethers.getContractFactory("MyToken", deployer);

  // Triển khai hợp đồng
  console.log("Deploying MyToken...");
  const token = await MyToken.deploy(initialSupply);

  // Chờ giao dịch triển khai hoàn tất
  const deployment = await token.waitForDeployment();
  const contractAddress = await token.getAddress();
  console.log("MyToken deployed to:", contractAddress);

  return contractAddress;
}

main()
  .then(() => process.exit(0))
  .catch((error) => {
    console.error(error);
    process.exit(1);
  });
```
- Đảm bảo đủ token testnet:
- Kiểm tra số dư ví trên chainscan-galileo.0g.ai.
- faucet faucet.0g.ai.

Triển khai hợp đồng:
```bash
npx hardhat run scripts/deploy.js --network galileo
```
Kết quả thực tế của bạn:
```
Deploying from: 0xxxx....
Deploying MyToken...
MyToken deployed to: 0xxxx....
```
Lưu địa chỉ hợp đồng 0xxxx....

- Kiểm tra trên chainscan:
-- Truy cập chainscan-galileo.0g.ai.
-- Xác nhận hợp đồng xuất hiện (xem giao dịch tạo hợp đồng).

## Lỗi mạng:
- Kiểm tra RPC URL:
```bash
curl -X POST -H "Content-Type: application/json" --data '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}' https://evmrpc-testnet.0g.ai/
```
# CHÚC CÁC BẠN THÀNH CÔNG TRIỂN KHAI ĐƯỢC TOKEN CHO RIÊNG MÌNH





