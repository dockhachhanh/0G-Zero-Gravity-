# THIẾT KẾ GAME: COIN FLIP (TUNG ĐỒNG XU)
## Mô tả game
- Tên game: Coin Flip
- Mục tiêu: Người chơi đặt cược một số tiền (token testnet) và chọn Head (mặt ngửa) hoặc Tail (mặt sấp). Hợp đồng sẽ tạo kết quả ngẫu nhiên, nếu đoán đúng, người chơi nhận gấp đôi số cược; nếu sai, mất số cược.
- Tính năng:
  - Đặt cược và chọn Head/Tail.
  - Kiểm tra kết quả và nhận thưởng.
  - Xem số dư ví trong hợp đồng.
- CLI: Người chơi (Alice và Bob) sẽ tương tác qua CLI để đặt cược và chơi.
- Môi trường: Triển khai trên 0G Galileo Testnet (Chain ID: 16601) với RPC https://evmrpc-testnet.0g.ai.
---
## Yêu cầu
- Hệ điều hành: Ubuntu 22.04 LTS (khuyến nghị), macOS, hoặc Windows (với Terminal/PowerShell).
- Công cụ:
  - Node.js và npm: Phiên bản 18.x hoặc 20.x.
  - Hardhat: Biên dịch và triển khai hợp đồng.
  - Bun: Chạy CLI (hoặc Node.js).
  - viem: Tương tác với hợp đồng.
- Ví Ethereum: 2 ví (Alice và Bob) với ít nhất 0.1 token testnet từ https://faucet.0g.ai/ (tài khoản X >10 follower).
- Kết nối internet: Tải gói, nạp token, tương tác với testnet.
---
## Cấu trúc dự án
```
coinflip-galileo/
├── contracts/              # Hợp đồng Solidity
│   └── CoinFlip.sol
├── scripts/                # Script triển khai
│   └── deploy.js
├── cli/                    # CLI để chơi game
│   ├── src/
│   │   ├── app.ts
│   │   └── index.ts
│   ├── lib/
│   │   ├── constants.ts
│   │   └── utils.ts
│   ├── artifacts/
│   │   └── contracts/
│   │       └── CoinFlip.sol/
│   │           └── CoinFlip.json
│   ├── .env
│   ├── .gitignore
│   ├── package.json
│   ├── package-lock.json
│   └── tsconfig.json
├── artifacts/              # File biên dịch
├── .env
├── .gitignore
├── hardhat.config.js
├── package.json
├── package-lock.json
```
---
## Hướng dẫn triển khai và chơi game
### Bước 1: Chuẩn bị môi trường
#### 1.1. Cài đặt Node.js và npm
- Kiểm tra:
```bash
node -v
npm -v
```
- Cài đặt nếu cần:
```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.2/install.sh | bash
source ~/.bashrc
nvm install 20
node -v  # Nên hiển thị v20.x.x
npm -v   # Nên hiển thị 10.x.x
```
#### 1.2. Cài đặt Bun
```bash
curl -fsSL https://bun.sh/install | bash
source ~/.bashrc
bun -v  # Nên hiển thị 1.x.x
```
#### 1.3. Cập nhật hệ thống (Ubuntu)
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install curl git wget jq build-essential -y
```
#### 1.4. Chuẩn bị ví Ethereum
- Tạo 2 ví (Alice và Bob) qua MetaMask.
- Lấy private key (không cần 0x).
- Nạp ít nhất 0.1 token testnet cho mỗi ví từ https://faucet.0g.ai/ (yêu cầu tài khoản X >10 follower).
- Kiểm tra số dư trên https://chainscan-galileo.0g.ai/.
#### 1.5. Tạo thư mục dự án
```bash
mkdir ~/coinflip-galileo
cd ~/coinflip-galileo
npm init -y
```
### Bước 2: Thiết lập dự án Hardhat
#### 2.1. Cài đặt Hardhat và phụ thuộc
```bash
npm install --save-dev hardhat @nomicfoundation/hardhat-toolbox @nomicfoundation/hardhat-verify dotenv
npm install viem
```
#### 2.2. Khởi tạo dự án Hardhat
```bash
npx hardhat
```
- Chọn Create a JavaScript project.
- Nhấn Enter để cài đặt phụ thuộc.
- Kiểm tra:
```bash
npx hardhat --version  # Nên hiển thị ≥ 2.17.0
```
#### 2.3. Cấu hình .env
- Tạo file .env:
```bash
nano .env
```
- Thêm:
```
RPC_URL=https://evmrpc-testnet.0g.ai
CHAIN_ID=16601
ALICE_PRIVATE_KEY=<alice-private-key>
BOB_PRIVATE_KEY=<bob-private-key>
CHAINSCAN_API_KEY=abc
```
- Thay ``` <alice-private-key> ``` và ``` <bob-private-key> ``` bằng khóa riêng.
- Lưu và thoát.
#### 2.4. Cấu hình .gitignore
- Tạo file .gitignore:
```bash
nano .gitignore
```
- Thêm:
```
node_modules
.env
artifacts
cache
```
#### 2.5. Cấu hình Hardhat
- Mở hardhat.config.js:
```bash
nano hardhat.config.js
```
- Thêm:
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
      chainId: 16601,
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
        chainId: 16601,
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
#### 2.6. Kiểm tra RPC
```bash
curl -X POST -H "Content-Type: application/json" --data '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}' https://evmrpc-testnet.0g.ai
```
- Kết quả mong đợi: ``` {"jsonrpc":"2.0","result":"0x1234","id":1} ```.
---
## Bước 3: Tạo và triển khai hợp đồng CoinFlip.sol
### 3.1. Tạo hợp đồng
- Tạo thư mục contracts/:
```bash
mkdir contracts
nano contracts/CoinFlip.sol
```
- Dán nội dung:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract CoinFlip {
    address public owner;
    mapping(address => uint256) public balances;

    event BetPlaced(address indexed player, bool choice, uint256 amount, bool won, uint256 payout);

    constructor() {
        owner = msg.sender;
    }

    function placeBet(bool _choice) external payable {
        require(msg.value > 0, "Bet amount must be greater than 0");

        // Generate pseudo-random result (0 = Tail, 1 = Head)
        bool result = (block.timestamp % 2) == 0;
        bool won = (_choice == result);
        uint256 payout = 0;

        if (won) {
            payout = msg.value * 2;
            require(address(this).balance >= payout, "Insufficient contract balance");
            balances[msg.sender] += payout;
        } else {
            balances[owner] += msg.value;
        }

        emit BetPlaced(msg.sender, _choice, msg.value, won, payout);
    }

    function withdraw() external {
        uint256 amount = balances[msg.sender];
        require(amount > 0, "No balance to withdraw");
        balances[msg.sender] = 0;
        (bool success, ) = msg.sender.call{value: amount}("");
        require(success, "Withdraw failed");
    }

    function getBalance() external view returns (uint256) {
        return balances[msg.sender];
    }

    // Allow owner to fund contract
    function fundContract() external payable {
        require(msg.sender == owner, "Only owner can fund");
        balances[owner] += msg.value;
    }
}
```
### Giải thích:
- Người chơi gọi placeBet(true) (Head) hoặc placeBet(false) (Tail) với số tiền cược (msg.value).
- Kết quả ngẫu nhiên dựa trên block.timestamp (đơn giản cho testnet).
- Nếu thắng, nhận gấp đôi cược; nếu thua, tiền cược vào ví owner.
- withdraw() để rút tiền thắng.
- getBalance() xem số dư trong hợp đồng.
- fundContract() cho phép owner nạp tiền vào hợp đồng để trả thưởng.
### 3.2. Biên dịch hợp đồng
```bash
npx hardhat compile
```
- Kết quả mong đợi:
```
Compiled 1 Solidity file successfully (evm target: istanbul).
```
### 3.3. Tạo script triển khai
- Tạo thư mục scripts/:
```bash
mkdir scripts
nano scripts/deploy.js
```
- Dán:
```javascript
const hre = require("hardhat");

async function main() {
  const [deployer] = await hre.ethers.getSigners();
  console.log("Deploying from:", deployer.address);

  const CoinFlip = await hre.ethers.getContractFactory("CoinFlip", deployer);
  console.log("Deploying CoinFlip...");
  const coinFlip = await CoinFlip.deploy();

  const contractAddress = await coinFlip.getAddress();
  console.log("CoinFlip deployed to:", contractAddress);

  return contractAddress;
}

main()
  .then(() => process.exit(0))
  .catch((error) => {
    console.error(error);
    process.exit(1);
  });
```
### 3.4. Triển khai hợp đồng
- Đảm bảo ví Alice có đủ token testnet:
```bash
npx hardhat run scripts/deploy.js --network galileo
```
- Kết quả mẫu:
```
Deploying from: 0xExxx
Deploying CoinFlip...
CoinFlip deployed to: 0x1234567890abcdef
```
- Lưu địa chỉ hợp đồng.
### 3.5. Kiểm tra trên Chainscan
- Truy cập https://chainscan-galileo.0g.ai/.
- Nhập địa chỉ hợp đồng để xác nhận.
---
## Bước 4: Xây dựng CLI để chơi game
### 4.1. Tạo thư mục CLI
```bash
mkdir cli
cd cli
npm init -y
npm install viem dotenv
npm install --save-dev typescript @types/node
```
- Tạo cấu trúc:
```bash
mkdir src lib artifacts artifacts/contracts artifacts/contracts/CoinFlip.sol
touch src/app.ts src/index.ts lib/constants.ts lib/utils.ts
touch .env .gitignore tsconfig.json
```
- 4.2. Copy file ABI
```bash
cp ../artifacts/contracts/CoinFlip.sol/CoinFlip.json artifacts/contracts/CoinFlip.sol/
```
### 4.3. Cấu hình .gitignore
- Mở cli/.gitignore:
```bash
nano .gitignore
```
- Thêm:
```
node_modules
.env
dist
```
### 4.4. Cấu hình .env
- Mở cli/.env:
```bash
nano .env
```
- Thêm:
```
RPC_URL=https://evmrpc-testnet.0g.ai
CHAIN_ID=16601
ALICE_PRIVATE_KEY=<alice-private-key>
BOB_PRIVATE_KEY=<bob-private-key>
CONTRACT_ADDRESS=<contract-address>
```
- Thay các giá trị bằng khóa riêng và địa chỉ hợp đồng.
### 4.5. Cấu hình package.json
- Mở cli/package.json:
```bash
nano package.json
```
- Thêm:
```json
{
  "name": "coinflip-cli",
  "version": "1.0.0",
  "license": "MIT",
  "type": "module",
  "scripts": {
    "dev": "bun run src/index.ts",
    "build": "tsc",
    "start": "node dist/index.js"
  },
  "dependencies": {
    "viem": "^2.28.0",
    "dotenv": "^16.4.7"
  },
  "devDependencies": {
    "typescript": "^5.6.3",
    "@types/node": "^22.7.6"
  }
}
```
### 4.6. Cài dependencies
```bash
npm install
```
### 4.7. Cấu hình tsconfig.json
- Mở cli/tsconfig.json:
```bash
nano tsconfig.json
```
- Thêm:
```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "ESNext",
    "moduleResolution": "node",
    "esModuleInterop": true,
    "forceConsistentCasingInFileNames": true,
    "strict": true,
    "skipLibCheck": true,
    "outDir": "./dist",
    "rootDir": "./src"
  },
  "include": ["src/**/*", "lib/**/*"]
}
```
### 4.8. Thêm lib/constants.ts
- Mở cli/lib/constants.ts:
```bash
nano lib/constants.ts
```
- Thêm:
```typescript
import { join } from 'path';

const CONTRACT_NAME = 'CoinFlip';
const CONTRACT_DIR = join(__dirname, '../../artifacts/contracts');

export { CONTRACT_NAME, CONTRACT_DIR };
```
### 4.9. Thêm lib/utils.ts
- Mở cli/lib/utils.ts:
```bash
nano lib/utils.ts
```
- Thêm:
```typescript
import fs from 'fs';
import { Abi, Address } from 'viem';

function readContractABI(abiFile: string): Abi {
  const abi = JSON.parse(fs.readFileSync(abiFile, 'utf8'));
  if (!abi.abi) {
    throw new Error('Invalid ABI file format');
  }
  return abi.abi;
}

export { readContractABI };
```
### 4.10. Thêm src/app.ts
- Mở cli/src/app.ts:
```bash
nano src/app.ts
```
- Thêm:
```typescript
import { Abi, Address, createWalletClient, createPublicClient, getContract, http } from 'viem';
import { privateKeyToAccount } from 'viem/accounts';
import { Chain } from 'viem/chains';

interface AppConfig {
  players: Array<{
    name: string;
    privateKey: string;
  }>;
  wallet: {
    chain: Chain;
    rpcUrl: string;
  };
  contract: {
    abi: Abi;
    address: Address;
  };
}

export class App {
  private config: AppConfig;
  private playerClients: Map<string, any> = new Map();
  private playerContracts: Map<string, any> = new Map();
  private publicClient: any;

  constructor(config: AppConfig) {
    this.config = config;
    this.publicClient = createPublicClient({
      chain: this.config.wallet.chain,
      transport: http(this.config.wallet.rpcUrl)
    });
  }

  async init() {
    for (const player of this.config.players) {
      const walletClient = createWalletClient({
        chain: this.config.wallet.chain,
        transport: http(this.config.wallet.rpcUrl),
        account: privateKeyToAccount(`0x${player.privateKey}` as `0x${string}`)
      });
      this.playerClients.set(player.name, walletClient);

      const contract = getContract({
        abi: this.config.contract.abi,
        address: this.config.contract.address,
        client: walletClient
      });
      this.playerContracts.set(player.name, contract);
    }
  }

  private getWalletClient(playerName: string) {
    const client = this.playerClients.get(playerName);
    if (!client) {
      throw new Error(`Wallet client for player ${playerName} not found`);
    }
    return client;
  }

  private getPlayerContract(playerName: string) {
    const contract = this.playerContracts.get(playerName);
    if (!contract) {
      throw new Error(`Contract for player ${playerName} not found`);
    }
    return contract;
  }

  async placeBet(playerName: string, choice: boolean, amount: bigint) {
    console.log(`- Player ${playerName} placing bet: ${choice ? 'Head' : 'Tail'}, Amount: ${amount} wei`);
    const contract = this.getPlayerContract(playerName);
    const hash = await contract.write.placeBet([choice], { value: amount, gas: 100000n });
    await this.publicClient.waitForTransactionReceipt({ hash });
  }

  async withdraw(playerName: string) {
    console.log(`- Player ${playerName} withdrawing balance`);
    const contract = this.getPlayerContract(playerName);
    const hash = await contract.write.withdraw({ gas: 100000n });
    await this.publicClient.waitForTransactionReceipt({ hash });
  }

  async getBalance(playerName: string) {
    console.log(`- Player ${playerName} checking balance`);
    const contract = this.getPlayerContract(playerName);
    const balance = await contract.read.getBalance();
    console.log(`- Player ${playerName} balance: ${balance} wei`);
    return balance;
  }
}
```
### 4.11. Thêm src/index.ts
- Mở cli/src/index.ts:
```bash
nano src/index.ts
```
- Thêm:
```typescript
import dotenv from 'dotenv';
import { join } from 'path';
import { defineChain } from 'viem';
import { CONTRACT_NAME, CONTRACT_DIR } from '../lib/constants';
import { readContractABI } from '../lib/utils';
import { App } from './app';

dotenv.config();

const galileo = defineChain({
  id: 16601,
  name: 'Galileo',
  network: 'galileo',
  nativeCurrency: {
    decimals: 18,
    name: '0G Token',
    symbol: '0G'
  },
  rpcUrls: {
    default: {
      http: ['https://evmrpc-testnet.0g.ai']
    }
  },
  blockExplorers: {
    default: {
      name: 'Chainscan',
      url: 'https://chainscan-galileo.0g.ai'
    }
  }
});

async function main() {
  if (!process.env.RPC_URL || !process.env.CHAIN_ID || !process.env.ALICE_PRIVATE_KEY || !process.env.BOB_PRIVATE_KEY || !process.env.CONTRACT_ADDRESS) {
    console.error('Please set all required environment variables.');
    process.exit(1);
  }

  const abiFile = join(
    CONTRACT_DIR,
    `${CONTRACT_NAME}.sol`,
    `${CONTRACT_NAME}.json`
  );

  const players = [
    { name: 'Alice', privateKey: process.env.ALICE_PRIVATE_KEY },
    { name: 'Bob', privateKey: process.env.BOB_PRIVATE_KEY }
  ];

  const app = new App({
    players,
    wallet: {
      chain: galileo,
      rpcUrl: process.env.RPC_URL
    },
    contract: {
      abi: readContractABI(abiFile),
      address: process.env.CONTRACT_ADDRESS as `0x${string}`
    }
  });

  await app.init();

  console.log('=== Game: Coin Flip ===');
  console.log('Round 1: Alice plays');
  await app.placeBet('Alice', true, BigInt(1e16)); // Bet 0.01 token
  await app.getBalance('Alice');

  console.log('\nRound 2: Bob plays');
  await app.placeBet('Bob', false, BigInt(1e16)); // Bet 0.01 token
  await app.getBalance('Bob');

  console.log('\nAlice withdraws winnings (if any)');
  await app.withdraw('Alice');
  await app.getBalance('Alice');

  console.log('\nBob withdraws winnings (if any)');
  await app.withdraw('Bob');
  await app.getBalance('Bob');
}

main();
```
### 4.12. Chạy CLI
- Trong thư mục cli/:
```bash
bun dev
```
- Kết quả mẫu:
```
=== Game: Coin Flip ===
Round 1: Alice plays
- Player Alice placing bet: Head, Amount: 10000000000000000 wei
- Player Alice checking balance
- Player Alice balance: 20000000000000000 wei

Round 2: Bob plays
- Player Bob placing bet: Tail, Amount: 10000000000000000 wei
- Player Bob checking balance
- Player Bob balance: 0 wei

Alice withdraws winnings (if any)
- Player Alice withdrawing balance
- Player Alice checking balance
- Player Alice balance: 0 wei

Bob withdraws winnings (if any)
- Player Bob withdrawing balance
- Player Bob checking balance
- Player Bob balance: 0 wei
```
- Giải thích: Kết quả phụ thuộc vào tính ngẫu nhiên của block.timestamp. Alice có thể thắng (nhận 0.02 token) hoặc thua (mất 0.01 token), tương tự với Bob.
---
## Bước 5: Xác minh hợp đồng
### 5.1. Flatten mã nguồn
```bash
npx hardhat flatten contracts/CoinFlip.sol > CoinFlipFlattened.sol
```
### 5.2. Xác minh thủ công
- Truy cập https://chainscan-galileo.0g.ai/.
- Nhập địa chỉ hợp đồng.
- Chuyển đến tab Contract, nhấn Verify and Publish.
- Điền:
  - Contract Address: Địa chỉ hợp đồng.
  - Contract Name: CoinFlip.
  - Compiler: 0.8.20.
  - Optimization: Enabled, Runs = 200.
  - EVM Version: istanbul.
  - License: MIT.
  - Contract Source Code: Dán nội dung CoinFlipFlattened.sol.
- Nhấn Verify.
---
