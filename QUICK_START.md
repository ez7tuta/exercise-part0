# Quick Start Guide - P2P Lottery Platform

This guide will help you get started quickly with building your P2P lottery prediction platform.

## Prerequisites

Before you begin, ensure you have:
- Node.js 18+ installed
- Git installed
- MetaMask wallet extension
- Basic understanding of JavaScript/TypeScript
- Basic understanding of blockchain concepts

## Step-by-Step Setup

### Phase 1: Environment Setup (Day 1)

#### 1. Create Project Directory

```bash
mkdir lottochain
cd lottochain
```

#### 2. Initialize Smart Contract Project

```bash
# Create smart contract directory
mkdir smart-contract
cd smart-contract

# Initialize Hardhat
npm init -y
npm install --save-dev hardhat @nomicfoundation/hardhat-toolbox
npx hardhat init
# Select "Create a TypeScript project"

# Install dependencies
npm install @openzeppelin/contracts
npm install dotenv
npm install @chainlink/contracts
```

#### 3. Configure Hardhat

Create `hardhat.config.ts`:

```typescript
import { HardhatUserConfig } from "hardhat/config";
import "@nomicfoundation/hardhat-toolbox";
import "dotenv/config";

const config: HardhatUserConfig = {
  solidity: {
    version: "0.8.19",
    settings: {
      optimizer: {
        enabled: true,
        runs: 200
      }
    }
  },
  networks: {
    // Polygon Mumbai Testnet
    mumbai: {
      url: process.env.MUMBAI_RPC_URL || "https://rpc-mumbai.maticvigil.com",
      accounts: process.env.PRIVATE_KEY ? [process.env.PRIVATE_KEY] : [],
      chainId: 80001
    },
    // Polygon Mainnet
    polygon: {
      url: process.env.POLYGON_RPC_URL || "https://polygon-rpc.com",
      accounts: process.env.PRIVATE_KEY ? [process.env.PRIVATE_KEY] : [],
      chainId: 137
    }
  },
  etherscan: {
    apiKey: process.env.POLYGONSCAN_API_KEY
  }
};

export default config;
```

#### 4. Create Environment Variables

Create `.env` file (never commit this!):

```bash
# Wallet
PRIVATE_KEY=your_private_key_here

# RPC URLs
MUMBAI_RPC_URL=https://polygon-mumbai.g.alchemy.com/v2/YOUR_API_KEY
POLYGON_RPC_URL=https://polygon-mainnet.g.alchemy.com/v2/YOUR_API_KEY

# Polygonscan
POLYGONSCAN_API_KEY=your_polygonscan_api_key

# Contract addresses (fill after deployment)
CONTRACT_ADDRESS=

# Redis
REDIS_URL=your_upstash_redis_url

# Encryption
ENCRYPTION_KEY=generate_with_openssl_rand_hex_32
```

Generate encryption key:
```bash
openssl rand -hex 32
```

#### 5. Create `.gitignore`

```bash
cat > .gitignore << EOF
# Dependencies
node_modules/
package-lock.json

# Environment
.env
.env.local

# Build outputs
artifacts/
cache/
dist/
build/

# IDE
.vscode/
.idea/

# OS
.DS_Store
Thumbs.db

# Hardhat
coverage/
typechain-types/
EOF
```

### Phase 2: Smart Contract Development (Day 2-4)

#### 1. Copy Smart Contract Code

Copy the complete smart contract from `IMPLEMENTATION_EXAMPLES.md` to:
```
smart-contract/contracts/P2PLotteryBetting.sol
```

#### 2. Write Tests

Create `test/P2PLottery.test.ts` and copy the test code from `IMPLEMENTATION_EXAMPLES.md`.

#### 3. Run Tests

```bash
npx hardhat test
```

Expected output:
```
  P2PLotteryBetting
    Betting
      ✓ Should allow users to place bets
      ✓ Should reject bets below minimum
      ✓ Should match bets between EVEN and ODD
    Finalization
      ✓ Should finalize round with winning number
    Claims
      ✓ Should allow winners to claim winnings
      ✓ Should refund unmatched bets

  6 passing (2s)
```

#### 4. Deploy to Testnet

Create `scripts/deploy.ts`:

```typescript
import { ethers } from "hardhat";

async function main() {
  console.log("Deploying P2PLotteryBetting...");
  
  const P2PLotteryBetting = await ethers.getContractFactory("P2PLotteryBetting");
  const contract = await P2PLotteryBetting.deploy();
  
  await contract.deployed();
  
  console.log("Contract deployed to:", contract.address);
  console.log("Waiting for block confirmations...");
  
  await contract.deployTransaction.wait(5);
  
  console.log("Verifying contract...");
  await ethers.run("verify:verify", {
    address: contract.address,
    constructorArguments: [],
  });
  
  console.log("✅ Deployment complete!");
  console.log("Contract address:", contract.address);
}

main()
  .then(() => process.exit(0))
  .catch((error) => {
    console.error(error);
    process.exit(1);
  });
```

Deploy:
```bash
npx hardhat run scripts/deploy.ts --network mumbai
```

Save the contract address in `.env`:
```bash
CONTRACT_ADDRESS=0x...
```

### Phase 3: Frontend Setup (Day 5-7)

#### 1. Create Next.js Project

```bash
cd ..
npx create-next-app@latest frontend --typescript --tailwind --app
cd frontend
```

#### 2. Install Dependencies

```bash
# Web3 libraries
npm install ethers wagmi viem @rainbow-me/rainbowkit

# UI libraries
npm install @headlessui/react @heroicons/react
npm install framer-motion

# Socket.io
npm install socket.io-client

# Forms
npm install react-hook-form @hookform/resolvers zod

# Utils
npm install date-fns
```

#### 3. Configure RainbowKit

Create `app/providers.tsx`:

```typescript
'use client';

import '@rainbow-me/rainbowkit/styles.css';
import { getDefaultWallets, RainbowKitProvider } from '@rainbow-me/rainbowkit';
import { configureChains, createConfig, WagmiConfig } from 'wagmi';
import { polygon, polygonMumbai } from 'wagmi/chains';
import { publicProvider } from 'wagmi/providers/public';

const { chains, publicClient } = configureChains(
  [polygonMumbai, polygon],
  [publicProvider()]
);

const { connectors } = getDefaultWallets({
  appName: 'LottoChain',
  projectId: 'YOUR_WALLETCONNECT_PROJECT_ID',
  chains
});

const wagmiConfig = createConfig({
  autoConnect: true,
  connectors,
  publicClient
});

export function Providers({ children }: { children: React.ReactNode }) {
  return (
    <WagmiConfig config={wagmiConfig}>
      <RainbowKitProvider chains={chains}>
        {children}
      </RainbowKitProvider>
    </WagmiConfig>
  );
}
```

#### 4. Update Root Layout

Update `app/layout.tsx`:

```typescript
import './globals.css';
import { Providers } from './providers';

export const metadata = {
  title: 'LottoChain - P2P Lottery Predictions',
  description: 'Decentralized peer-to-peer lottery prediction platform',
};

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="en">
      <body>
        <Providers>{children}</Providers>
      </body>
    </html>
  );
}
```

#### 5. Create Contract Hook

Create `hooks/useContract.ts`:

```typescript
import { useContractRead, useContractWrite, useWaitForTransaction } from 'wagmi';
import contractABI from '../contracts/abi.json';

const CONTRACT_ADDRESS = process.env.NEXT_PUBLIC_CONTRACT_ADDRESS!;

export function useContract() {
  // Place bet
  const { 
    data: placeBetData, 
    write: placeBet,
    isLoading: placeBetLoading 
  } = useContractWrite({
    address: CONTRACT_ADDRESS,
    abi: contractABI,
    functionName: 'placeBet',
  });
  
  // Get round info
  const { 
    data: roundInfo,
    isLoading: roundInfoLoading,
    refetch: refetchRoundInfo
  } = useContractRead({
    address: CONTRACT_ADDRESS,
    abi: contractABI,
    functionName: 'getRoundInfo',
    args: [1], // Current round
  });
  
  // Get user bet
  function useUserBet(roundId: number, address: `0x${string}`) {
    return useContractRead({
      address: CONTRACT_ADDRESS,
      abi: contractABI,
      functionName: 'getUserBet',
      args: [roundId, address],
    });
  }
  
  return {
    placeBet,
    placeBetLoading,
    roundInfo,
    roundInfoLoading,
    refetchRoundInfo,
    useUserBet
  };
}
```

#### 6. Create Betting Interface Component

Copy the `BettingInterface.tsx` component from `IMPLEMENTATION_EXAMPLES.md` to `components/BettingInterface.tsx`.

#### 7. Create Home Page

Update `app/page.tsx`:

```typescript
import { BettingInterface } from '@/components/BettingInterface';

export default function Home() {
  return (
    <main className="min-h-screen bg-gradient-to-br from-blue-50 to-purple-50 py-8">
      <BettingInterface />
    </main>
  );
}
```

#### 8. Generate Contract ABI

After deploying your smart contract, copy the ABI:

```bash
cd ../smart-contract
mkdir -p ../frontend/contracts
cat artifacts/contracts/P2PLotteryBetting.sol/P2PLotteryBetting.json | jq '.abi' > ../frontend/contracts/abi.json
```

#### 9. Run Frontend

```bash
cd ../frontend
npm run dev
```

Open http://localhost:3000

### Phase 4: Backend & Real-time (Day 8-9)

#### 1. Set Up Socket.io Server

Create `backend/` directory:

```bash
cd ..
mkdir backend
cd backend
npm init -y
npm install express socket.io cors dotenv
npm install --save-dev typescript @types/express @types/node ts-node
```

Create `server.ts`:

```typescript
import express from 'express';
import { createServer } from 'http';
import { Server } from 'socket.io';
import cors from 'cors';
import dotenv from 'dotenv';

dotenv.config();

const app = express();
app.use(cors());

const httpServer = createServer(app);
const io = new Server(httpServer, {
  cors: {
    origin: process.env.FRONTEND_URL || 'http://localhost:3000',
    credentials: true
  }
});

io.on('connection', (socket) => {
  console.log('Client connected:', socket.id);
  
  socket.on('subscribe:round', (roundId) => {
    socket.join(`round:${roundId}`);
    console.log(`Client ${socket.id} subscribed to round ${roundId}`);
  });
  
  socket.on('disconnect', () => {
    console.log('Client disconnected:', socket.id);
  });
});

// Health check
app.get('/health', (req, res) => {
  res.json({ status: 'ok', timestamp: Date.now() });
});

const PORT = process.env.PORT || 3001;
httpServer.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});

export { io };
```

#### 2. Set Up Redis

Sign up for Upstash Redis (free tier): https://upstash.com

Get your Redis URL and add to `.env`:
```
REDIS_URL=your_upstash_redis_url
```

Install Redis client:
```bash
npm install @upstash/redis
```

#### 3. Deploy to Railway

1. Create account on Railway.app
2. Create new project
3. Deploy from GitHub
4. Add environment variables
5. Get deployment URL

### Phase 5: Testing & Launch (Day 10)

#### 1. End-to-End Testing

Test complete flow:
1. ✅ Connect wallet
2. ✅ Place bet
3. ✅ See real-time updates
4. ✅ Wait for round to lock
5. ✅ Submit oracle result (manually for testing)
6. ✅ Claim winnings

#### 2. Deploy Frontend to Vercel

```bash
cd frontend
npm run build
vercel deploy --prod
```

#### 3. Configure Cloudflare

1. Add your domain to Cloudflare
2. Update DNS to point to Vercel
3. Enable security features:
   - Security Level: High
   - Bot Fight Mode: On
   - Rate Limiting: Enabled
   - WAF: Enabled

#### 4. Go Live!

Your P2P lottery platform is now live! 🎉

## Development Workflow

### Daily Development

```bash
# Terminal 1 - Smart contract (when making contract changes)
cd smart-contract
npx hardhat test --watch

# Terminal 2 - Frontend
cd frontend
npm run dev

# Terminal 3 - Backend
cd backend
npm run dev
```

### Making Contract Changes

```bash
cd smart-contract

# 1. Edit contract
nano contracts/P2PLotteryBetting.sol

# 2. Run tests
npx hardhat test

# 3. Deploy to testnet
npx hardhat run scripts/deploy.ts --network mumbai

# 4. Update ABI in frontend
cat artifacts/contracts/P2PLotteryBetting.sol/P2PLotteryBetting.json | jq '.abi' > ../frontend/contracts/abi.json

# 5. Update contract address in frontend .env
echo "NEXT_PUBLIC_CONTRACT_ADDRESS=0xNEW_ADDRESS" >> ../frontend/.env.local
```

## Common Issues & Solutions

### Issue: Transaction Fails with "Betting closed"

**Solution**: The round's lock time has passed. Create a new round with future time:
```bash
npx hardhat run scripts/createRound.js --network mumbai
```

### Issue: "Insufficient funds for gas"

**Solution**: Get testnet MATIC from faucet:
- Mumbai faucet: https://faucet.polygon.technology/

### Issue: Contract not verified on Polygonscan

**Solution**: Manually verify:
```bash
npx hardhat verify --network mumbai CONTRACT_ADDRESS
```

### Issue: Socket.io not connecting

**Solution**: Check CORS settings in backend and ensure frontend URL is whitelisted.

## Next Steps

After completing the basic setup:

1. **Add More Prediction Types**: Implement HIGH/LOW, combined predictions
2. **Oracle Integration**: Set up Chainlink oracle for automatic lottery results
3. **User Dashboard**: Create page showing betting history, winnings, etc.
4. **Mobile App**: Build React Native app using same smart contract
5. **Analytics**: Add charts and statistics
6. **Notifications**: Email/push notifications for round results
7. **Referral System**: Add referral rewards
8. **Multi-lottery**: Support multiple lottery types (Powerball, Mega Millions, etc.)

## Resources

### Documentation
- Hardhat: https://hardhat.org/docs
- Ethers.js: https://docs.ethers.org
- Wagmi: https://wagmi.sh
- RainbowKit: https://rainbowkit.com
- Next.js: https://nextjs.org/docs
- Socket.io: https://socket.io/docs

### Tools
- Polygon Faucet: https://faucet.polygon.technology/
- Polygonscan: https://polygonscan.com
- Upstash Redis: https://upstash.com
- Vercel: https://vercel.com
- Railway: https://railway.app

### Security
- OpenZeppelin: https://docs.openzeppelin.com
- Slither: https://github.com/crytic/slither
- Mythril: https://github.com/ConsenSys/mythril

## Support

If you encounter issues:
1. Check the documentation
2. Review error messages carefully
3. Test on Mumbai testnet first
4. Use console.log for debugging
5. Ask in Web3 developer communities

## Congratulations! 🎉

You now have a working P2P lottery prediction platform! Start testing, iterating, and building your user base.

Remember:
- ✅ Test thoroughly on testnet
- ✅ Get contract audited before mainnet
- ✅ Start with small bets
- ✅ Implement all security measures
- ✅ Monitor everything closely
- ✅ Gather user feedback
- ✅ Iterate and improve

Good luck with your project! 🚀
