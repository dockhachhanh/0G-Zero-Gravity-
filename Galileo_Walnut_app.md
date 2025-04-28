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
---
## Bước 3: Tạo và triển khai hợp đồng Walnut.sol
### Tạo hợp đồng:
Tạo thư mục contracts/:
```bash
mkdir contracts
nano contracts/Walnut.sol
```
Dán nội dung:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract Walnut {
    uint256 public initialShellStrength;
    uint256 public shellStrength;
    uint256 public round;
    uint256 public initialKernel;
    uint256 public kernel;
    mapping(uint256 => mapping(address => uint256)) public hitsPerRound;

    event Hit(uint256 indexed round, address indexed hitter, uint256 remaining);
    event Shake(uint256 indexed round, address indexed shaker);
    event Reset(uint256 indexed newRound, uint256 shellStrength);

    constructor(uint256 _shellStrength, uint256 _kernel) {
        initialShellStrength = _shellStrength;
        shellStrength = _shellStrength;
        initialKernel = _kernel;
        kernel = _kernel;
        round = 1;
    }

    function getShellStrength() public view returns (uint256) {
        return shellStrength;
    }

    function hit() public requireIntact {
        shellStrength--;
        hitsPerRound[round][msg.sender]++;
        emit Hit(round, msg.sender, shellStrength);
    }

    function shake(uint256 _numShakes) public requireIntact {
        kernel += _numShakes;
        emit Shake(round, msg.sender);
    }

    function reset() public requireCracked {
        shellStrength = initialShellStrength;
        kernel = initialKernel;
        round++;
        emit Reset(round, shellStrength);
    }

    function look() public view requireCracked onlyContributor returns (uint256) {
        return kernel;
    }

    modifier requireCracked() {
        require(shellStrength == 0, "SHELL_INTACT");
        _;
    }

    modifier requireIntact() {
        require(shellStrength > 0, "SHELL_ALREADY_CRACKED");
        _;
    }

    modifier onlyContributor() {
        require(hitsPerRound[round][msg.sender] > 0, "NOT_A_CONTRIBUTOR");
        _;
    }
}
```
Lưu và thoát.
### Biên dịch hợp đồng:
```bash
npx hardhat compile
```
### Kết quả mong đợi:
```
Compiled 1 Solidity file successfully (evm target: istanbul).
```
File biên dịch lưu trong artifacts/.
### Tạo script triển khai:
Tạo thư mục scripts/:
```bash
mkdir scripts
nano scripts/deploy.js
```
Dán nội dung:
```javascript
const hre = require("hardhat");

async function main() {
  const [deployer] = await hre.ethers.getSigners();
  console.log("Deploying from:", deployer.address);

  const initialShellStrength = 3;
  const initialKernel = 0;

  const Walnut = await hre.ethers.getContractFactory("Walnut", deployer);
  console.log("Deploying Walnut...");
  const walnut = await Walnut.deploy(initialShellStrength, initialKernel);

  const contractAddress = await walnut.getAddress();
  console.log("Walnut deployed to:", contractAddress);

  return contractAddress;
}

main()
  .then(() => process.exit(0))
  .catch((error) => {
    console.error(error);
    process.exit(1);
  });
```
Lưu và thoát.
### Triển khai hợp đồng:
Đảm bảo ví của Alice (hoặc Bob) có đủ token testnet từ https://faucet.0g.ai/.
Chạy:
```bash
npx hardhat run scripts/deploy.js --network galileo
```
Kết quả mẫu:
```
Deploying from: 0xExxx
Deploying Walnut...
Walnut deployed to: 0x4E9E32e7C0B7E15Cf34397AF16fdb7f2E4bf666b
```
Lưu địa chỉ hợp đồng (ví dụ: 0x4E9E32e7C0B7E15Cf34397AF16fdb7f2E4bf666b).
### Kiểm tra trên Chainscan:
Truy cập https://chainscan-galileo.0g.ai/.
Nhập địa chỉ hợp đồng để xác nhận giao dịch tạo hợp đồng.
---
## Bước 4: Xây dựng CLI
### Tạo thư mục CLI:
```bash
mkdir cli
cd cli
npm init -y
npm install viem dotenv
npm install --save-dev typescript @types/node
```
Tạo cấu trúc thư mục:
```bash
mkdir src lib artifacts artifacts/contracts artifacts/contracts/Walnut.sol
touch src/app.ts src/index.ts lib/constants.ts lib/utils.ts
touch .env .gitignore tsconfig.json
```
### Copy file ABI:
Copy ``` Walnut.json ``` từ thư mục gốc:
```bash
cp ../artifacts/contracts/Walnut.sol/Walnut.json artifacts/contracts/Walnut.sol/
```
Kiểm tra:
```bash
ls artifacts/contracts/Walnut.sol/
```
Nên thấy ``` Walnut.json ```.

Cấu hình ``` .gitignore ```:
Mở ``` cli/.gitignore ```:
```bash
nano .gitignore
```
Thêm:
```
node_modules
.env
dist
```
Lưu và thoát.
### Cấu hình * .env *:
Mở ``` cli/.env ```:
```bash
nano .env
```
Thêm:
```
RPC_URL=https://evmrpc-testnet.0g.ai/
CHAIN_ID=80087
ALICE_PRIVATE_KEY=<alice-private-key>
BOB_PRIVATE_KEY=<bob-private-key>
CONTRACT_ADDRESS=<contract-address>
```
Thay các giá trị bằng khóa riêng và địa chỉ hợp đồng thực tế.
Lưu và thoát.
### Cấu hình package.json:
Mở ``` cli/package.json ```:
```bash
nano package.json
```
Thay bằng:
```json
{
  "name": "walnut-cli",
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
Lưu và thoát.
Cài dependencies
```
npm install
```
### Cấu hình tsconfig.json:
- Mở ``` cli/tsconfig.json```
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
Lưu và thoát.
### Thêm lib/constants.ts:
Mở ``` cli/lib/constants.ts```:
```bash
nano lib/constants.ts
```
Thêm:
```typescript

import { join } from 'path';

const CONTRACT_NAME = 'Walnut';
const CONTRACT_DIR = join(__dirname, '../../artifacts/contracts');

export { CONTRACT_NAME, CONTRACT_DIR };
```
Lưu và thoát.
### Thêm lib/utils.ts:
Mở ``` cli/lib/utils.ts```:
```bash

nano lib/utils.ts
```
Thêm:
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
Lưu và thoát.
### Thêm src/app.ts:
Mở ``` cli/src/app.ts```:
```bash
nano src/app.ts
```
Thêm:
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

  async reset(playerName: string) {
    console.log(`- Player ${playerName} writing reset()`);
    const contract = this.getPlayerContract(playerName);
    const hash = await contract.write.reset({ gas: 100000n });
    await this.publicClient.waitForTransactionReceipt({ hash });
  }

  async shake(playerName: string, numShakes: number) {
    console.log(`- Player ${playerName} writing shake()`);
    const contract = this.getPlayerContract(playerName);
    const hash = await contract.write.shake([numShakes], { gas: 50000n });
    await this.publicClient.waitForTransactionReceipt({ hash });
  }

  async hit(playerName: string) {
    console.log(`- Player ${playerName} writing hit()`);
    const contract = this.getPlayerContract(playerName);
    const hash = await contract.write.hit({ gas: 100000n });
    await this.publicClient.waitForTransactionReceipt({ hash });
  }

  async look(playerName: string) {
    console.log(`- Player ${playerName} reading look()`);
    const contract = this.getPlayerContract(playerName);
    const result = await contract.read.look();
    console.log(`- Player ${playerName} sees number:`, result);
  }
}
```
Lưu và thoát.
### Thêm src/index.ts:
Mở ``` cli/src/index.ts```:
```bash
nano src/index.ts
```
Thêm:
```typescript
import dotenv from 'dotenv';
import { join } from 'path';
import { defineChain } from 'viem';
import { CONTRACT_NAME, CONTRACT_DIR } from '../lib/constants';
import { readContractABI } from '../lib/utils';
import { App } from './app';

dotenv.config();

const galileo = defineChain({
  id: 80087,
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

  console.log('=== Round 1 ===');
  await app.reset('Alice');
  await app.shake('Alice', 2);
  await app.hit('Alice');
  await app.shake('Alice', 4);
  await app.hit('Alice');
  await app.shake('Alice', 1);
  await app.hit('Alice');
  await app.look('Alice');

  console.log('=== Round 2 ===');
  await app.reset('Bob');
  await app.hit('Bob');
  await app.shake('Bob', 1);
  await app.hit('Bob');
  await app.shake('Bob', 1);
  await app.hit('Bob');
  await app.look('Bob');

  console.log('=== Testing Access Control ===');
  console.log("Attempting Alice's look() in Bob's round (should revert)");
  try {
    await app.look('Alice');
    console.error('❌ Expected look() to revert but it succeeded');
    process.exit(1);
  } catch (error) {
    console.log('✅ Received expected revert');
  }
}

main();
```
Lưu và thoát.
### Chạy CLI:
Trong thư mục ``` cli/```:
```bash
bun dev
```
Kết quả mong đợi:
```
=== Round 1 ===
- Player Alice writing reset()
- Player Alice writing shake()
- Player Alice writing hit()
- Player Alice writing shake()
- Player Alice writing hit()
- Player Alice writing shake()
- Player Alice writing hit()
- Player Alice reading look()
- Player Alice sees number: 7
=== Round 2 ===
- Player Bob writing reset()
- Player Bob writing hit()
- Player Bob writing shake()
- Player Bob writing hit()
- Player Bob writing shake()
- Player Bob writing hit()
- Player Bob reading look()
- Player Bob sees number: 2
=== Testing Access Control ===
Attempting Alice's look() in Bob's round (should revert)
✅ Received expected revert
```
---
## Bước 5: Xác minh hợp đồng
### Flatten mã nguồn:
- Trong thư mục gốc (~/walnut-galileo):
```bash
npx hardhat flatten contracts/Walnut.sol > WalnutFlattened.sol
```
### Xác minh thủ công:
- Truy cập https://chainscan-galileo.0g.ai/.
- Nhập địa chỉ hợp đồng (ví dụ: 0x4E9E32e7C0B7E15Cf34397AF16fdb7f2E4bf666b).
- Chuyển đến tab Contract, nhấn Verify and Publish.
- Điền thông tin:
+ Contract Address: Địa chỉ hợp đồng.
+ Contract Name: Walnut.
+ Compiler: 0.8.20.
+ Optimization: Enabled, Runs = 200.
+ EVM Version: istanbul.
+ License: MIT.
+ Contract Source Code: Dán nội dung WalnutFlattened.sol hoặc tải lên file.
- Nhấn Verify, đợi thông báo Thành công.
---
## Bước 6: Kiểm tra giao dịch
### Xem giao dịch trên Chainscan:
- Truy cập ``` https://chainscan-galileo.0g.ai/ ```.
- Nhập địa chỉ hợp đồng hoặc địa chỉ ví của Alice/Bob.
- Xác nhận các giao dịch ``` hit, shake, reset, và look ``` được ghi nhận.
- Kiểm tra số dư ví: Đảm bảo ví của Alice và Bob vẫn có đủ 0.1 token testnet để thực hiện giao dịch.
- Nếu thiếu, nạp thêm từ https://faucet.0g.ai/.
