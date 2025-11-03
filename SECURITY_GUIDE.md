# Security Guide - P2P Lottery Platform

## 12 Essential Security Techniques Implementation

This guide provides detailed implementation for each of the 12 security techniques mentioned in the main documentation, with code examples and best practices.

---

## 1. Smart Contract Security - Reentrancy Protection

### What is Reentrancy?
A reentrancy attack occurs when an external contract calls back into the calling contract before the first execution is complete, potentially draining funds.

### Implementation

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract SecureP2PLottery {
    // State variable to track if function is being executed
    bool private locked;
    
    // Modifier to prevent reentrancy
    modifier noReentrant() {
        require(!locked, "ReentrancyGuard: reentrant call");
        locked = true;
        _;
        locked = false;
    }
    
    // Use with all functions that transfer funds
    function claimWinnings(uint256 roundId) external noReentrant {
        // 1. Check conditions
        require(rounds[roundId].isFinalized, "Round not finalized");
        Bet storage userBet = rounds[roundId].userBets[msg.sender];
        require(!userBet.claimed, "Already claimed");
        
        // 2. Update state BEFORE external call
        userBet.claimed = true;
        uint256 amount = calculateWinnings(roundId, msg.sender);
        
        // 3. External call last (CEI pattern: Checks, Effects, Interactions)
        (bool success, ) = payable(msg.sender).call{value: amount}("");
        require(success, "Transfer failed");
        
        emit WinningsClaimed(roundId, msg.sender, amount);
    }
    
    // Alternative: Use OpenZeppelin's ReentrancyGuard
    // import "@openzeppelin/contracts/security/ReentrancyGuard.sol";
    // contract SecureP2PLottery is ReentrancyGuard {
    //     function claimWinnings() external nonReentrant { ... }
    // }
}
```

### Best Practices
- ✅ Always use Checks-Effects-Interactions pattern
- ✅ Update state before external calls
- ✅ Use OpenZeppelin's ReentrancyGuard for production
- ✅ Test thoroughly with attack simulations

---

## 2. Access Control - Role-Based Permissions

### Implementation

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/access/AccessControl.sol";

contract P2PLotteryWithRoles is AccessControl {
    // Define roles
    bytes32 public constant ADMIN_ROLE = keccak256("ADMIN_ROLE");
    bytes32 public constant ORACLE_ROLE = keccak256("ORACLE_ROLE");
    bytes32 public constant OPERATOR_ROLE = keccak256("OPERATOR_ROLE");
    
    constructor() {
        // Grant admin role to deployer
        _grantRole(DEFAULT_ADMIN_ROLE, msg.sender);
        _grantRole(ADMIN_ROLE, msg.sender);
    }
    
    // Only admin can add oracles
    function addOracle(address oracleAddress) external onlyRole(ADMIN_ROLE) {
        _grantRole(ORACLE_ROLE, oracleAddress);
    }
    
    // Only oracle can submit results
    function finalizeRound(uint256 roundId, uint8 result) 
        external 
        onlyRole(ORACLE_ROLE) 
    {
        require(!rounds[roundId].isFinalized, "Already finalized");
        rounds[roundId].isFinalized = true;
        rounds[roundId].winningNumber = result;
        emit RoundFinalized(roundId, result);
    }
    
    // Only operator can create rounds
    function createRound(uint256 drawTime) 
        external 
        onlyRole(OPERATOR_ROLE) 
        returns (uint256) 
    {
        currentRoundId++;
        rounds[currentRoundId] = Round({
            roundId: currentRoundId,
            drawTime: drawTime,
            lockTime: drawTime - 15 minutes,
            isFinalized: false,
            winningNumber: 0
        });
        return currentRoundId;
    }
    
    // Emergency pause (admin only)
    function pause() external onlyRole(ADMIN_ROLE) {
        _pause();
    }
    
    function unpause() external onlyRole(ADMIN_ROLE) {
        _unpause();
    }
}
```

### Multi-Signature Control

```solidity
// For critical operations, use multi-sig
contract MultiSigP2PLottery {
    address[] public admins;
    mapping(bytes32 => uint256) public confirmations;
    uint256 public requiredConfirmations = 2;
    
    function executeWithMultiSig(
        bytes32 operationHash,
        address target,
        bytes memory data
    ) external {
        require(isAdmin(msg.sender), "Not admin");
        
        confirmations[operationHash]++;
        
        if (confirmations[operationHash] >= requiredConfirmations) {
            (bool success, ) = target.call(data);
            require(success, "Execution failed");
            delete confirmations[operationHash];
        }
    }
}
```

---

## 3. Input Validation - Comprehensive Checks

### Smart Contract Validation

```solidity
contract ValidatedP2PLottery {
    // Constants for validation
    uint256 public constant MIN_BET = 0.01 ether;
    uint256 public constant MAX_BET = 100 ether;
    uint256 public constant MAX_ROUNDS = 1000;
    
    function placeBet(
        uint256 roundId, 
        PredictionType prediction
    ) external payable {
        // 1. Validate amount
        require(msg.value >= MIN_BET, "Bet too small");
        require(msg.value <= MAX_BET, "Bet too large");
        
        // 2. Validate round
        require(roundId > 0 && roundId <= currentRoundId, "Invalid round ID");
        require(roundId <= MAX_ROUNDS, "Round ID too large");
        
        // 3. Validate prediction type
        require(uint8(prediction) < 8, "Invalid prediction type");
        
        // 4. Validate timing
        require(block.timestamp < rounds[roundId].lockTime, "Betting closed");
        require(!rounds[roundId].isFinalized, "Round finalized");
        
        // 5. Validate user hasn't bet already
        require(
            rounds[roundId].userBets[msg.sender].amount == 0, 
            "Already placed bet"
        );
        
        // 6. Validate round is active
        require(rounds[roundId].status == RoundStatus.OPEN, "Round not open");
        
        // ... place bet logic
    }
    
    // Validate address is not zero
    modifier validAddress(address addr) {
        require(addr != address(0), "Invalid address");
        require(addr != address(this), "Cannot be contract address");
        _;
    }
    
    // Validate amount is reasonable
    modifier validAmount(uint256 amount) {
        require(amount > 0, "Amount must be positive");
        require(amount <= type(uint128).max, "Amount too large");
        _;
    }
}
```

### Backend Validation

```typescript
// api/validation.ts
import { z } from 'zod';

// Zod schemas for validation
export const placeBetSchema = z.object({
  roundId: z.number().int().positive().max(1000),
  prediction: z.enum(['EVEN', 'ODD', 'HIGH', 'LOW', 'EVEN_LOW', 'ODD_HIGH', 'EVEN_HIGH', 'ODD_LOW']),
  amount: z.string().regex(/^\d+\.?\d*$/).refine(
    (val) => {
      const num = parseFloat(val);
      return num >= 0.01 && num <= 100;
    },
    { message: "Amount must be between 0.01 and 100 ETH" }
  ),
  walletAddress: z.string().regex(/^0x[a-fA-F0-9]{40}$/),
  signature: z.string().min(132).max(132),
});

export const roundIdSchema = z.number().int().positive().max(1000);

// Validation middleware
export async function validateRequest(req: any, schema: z.ZodSchema) {
  try {
    const validated = await schema.parseAsync(req.body);
    return { success: true, data: validated };
  } catch (error) {
    if (error instanceof z.ZodError) {
      return { 
        success: false, 
        errors: error.errors.map(e => ({
          field: e.path.join('.'),
          message: e.message
        }))
      };
    }
    return { success: false, errors: [{ message: 'Validation failed' }] };
  }
}

// Usage in API route
export async function POST(req: Request) {
  const validation = await validateRequest(req, placeBetSchema);
  
  if (!validation.success) {
    return Response.json({ errors: validation.errors }, { status: 400 });
  }
  
  const { roundId, prediction, amount, walletAddress } = validation.data;
  // ... process bet
}
```

### Frontend Validation

```typescript
// components/BettingForm.tsx
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';

export function BettingForm() {
  const {
    register,
    handleSubmit,
    formState: { errors }
  } = useForm({
    resolver: zodResolver(placeBetSchema)
  });
  
  const onSubmit = async (data: any) => {
    // Additional client-side checks
    if (!window.ethereum) {
      alert('Please install MetaMask');
      return;
    }
    
    // Check wallet balance
    const provider = new ethers.providers.Web3Provider(window.ethereum);
    const balance = await provider.getBalance(data.walletAddress);
    
    if (balance.lt(ethers.utils.parseEther(data.amount))) {
      alert('Insufficient balance');
      return;
    }
    
    // Submit to API
    await placeBet(data);
  };
  
  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input 
        {...register('amount')} 
        type="number" 
        step="0.01"
        min="0.01"
        max="100"
        placeholder="Bet amount (ETH)"
      />
      {errors.amount && <span>{errors.amount.message}</span>}
      
      <button type="submit">Place Bet</button>
    </form>
  );
}
```

---

## 4. Oracle Security - Multiple Sources & Consensus

### Chainlink Integration with Multiple Oracles

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@chainlink/contracts/src/v0.8/ChainlinkClient.sol";

contract OracleSecureP2PLottery is ChainlinkClient {
    using Chainlink for Chainlink.Request;
    
    // Multiple oracle addresses
    address[] public oracles;
    mapping(address => bool) public isOracle;
    
    // Consensus tracking
    mapping(uint256 => mapping(address => uint8)) public oracleResults;
    mapping(uint256 => mapping(uint8 => uint256)) public resultVotes;
    
    uint256 public constant REQUIRED_CONSENSUS = 2; // 2 out of 3 oracles
    
    // Chainlink job IDs
    bytes32 private jobId;
    uint256 private fee;
    
    constructor() {
        // Initialize Chainlink
        setChainlinkToken(0x326C977E6efc84E512bB9C30f76E30c160eD06FB); // Link token on Polygon Mumbai
        setChainlinkOracle(0x40193c8518BB267228Fc409a613bDbD8eC5a97b3);
        jobId = "7d80a6386ef543a3abb52817f6707e3b";
        fee = 0.1 * 10 ** 18; // 0.1 LINK
        
        // Add oracle addresses
        addOracle(0x1234...); // Oracle 1
        addOracle(0x5678...); // Oracle 2
        addOracle(0x9abc...); // Oracle 3
    }
    
    function addOracle(address oracle) public onlyOwner {
        require(!isOracle[oracle], "Already oracle");
        oracles.push(oracle);
        isOracle[oracle] = true;
    }
    
    // Request lottery result from Chainlink
    function requestLotteryResult(uint256 roundId, string memory apiUrl) 
        public 
        onlyOwner 
        returns (bytes32 requestId) 
    {
        Chainlink.Request memory request = buildChainlinkRequest(
            jobId,
            address(this),
            this.fulfill.selector
        );
        
        // Set the URL to fetch lottery results
        request.add("get", apiUrl);
        request.add("path", "data.winningNumber");
        
        // Send request
        return sendChainlinkRequest(request, fee);
    }
    
    // Callback function for Chainlink
    function fulfill(bytes32 _requestId, uint8 _result) 
        public 
        recordChainlinkFulfillment(_requestId) 
    {
        uint256 roundId = getCurrentRoundId();
        submitOracleResult(roundId, _result);
    }
    
    // Oracle submits result
    function submitOracleResult(uint256 roundId, uint8 result) public {
        require(isOracle[msg.sender], "Not authorized oracle");
        require(!rounds[roundId].isFinalized, "Already finalized");
        require(oracleResults[roundId][msg.sender] == 0, "Already submitted");
        
        // Record oracle's result
        oracleResults[roundId][msg.sender] = result;
        resultVotes[roundId][result]++;
        
        emit OracleResultSubmitted(roundId, msg.sender, result);
        
        // Check for consensus
        if (resultVotes[roundId][result] >= REQUIRED_CONSENSUS) {
            finalizeRoundWithConsensus(roundId, result);
        }
    }
    
    // Finalize with consensus
    function finalizeRoundWithConsensus(uint256 roundId, uint8 result) internal {
        require(!rounds[roundId].isFinalized, "Already finalized");
        
        rounds[roundId].isFinalized = true;
        rounds[roundId].winningNumber = result;
        
        emit RoundFinalized(roundId, result);
    }
    
    // Dispute mechanism (if oracles disagree)
    function disputeResult(uint256 roundId) external onlyOwner {
        require(rounds[roundId].isFinalized, "Not finalized");
        require(block.timestamp < rounds[roundId].drawTime + 1 hours, "Dispute period ended");
        
        // Reset results and request new submissions
        rounds[roundId].isFinalized = false;
        
        // Clear oracle results
        for (uint i = 0; i < oracles.length; i++) {
            delete oracleResults[roundId][oracles[i]];
        }
        
        emit ResultDisputed(roundId);
    }
    
    event OracleResultSubmitted(uint256 indexed roundId, address indexed oracle, uint8 result);
    event ResultDisputed(uint256 indexed roundId);
}
```

### Manual Oracle Verification

```typescript
// backend/oracle/verifyResults.ts
import axios from 'axios';

interface LotteryResult {
  drawDate: string;
  winningNumbers: number[];
  megaBall?: number;
  powerBall?: number;
}

// Multiple API sources for verification
const LOTTERY_APIS = {
  primary: 'https://api.lotteryusa.com/powerball/latest',
  secondary: 'https://data.ny.gov/api/powerball/latest',
  tertiary: 'https://api.lottery.com/powerball/draws/latest'
};

export async function fetchLotteryResults(): Promise<LotteryResult | null> {
  const results: LotteryResult[] = [];
  
  // Fetch from all sources
  for (const [source, url] of Object.entries(LOTTERY_APIS)) {
    try {
      const response = await axios.get(url, { timeout: 5000 });
      results.push(response.data);
      console.log(`Fetched from ${source}:`, response.data);
    } catch (error) {
      console.error(`Failed to fetch from ${source}:`, error);
    }
  }
  
  // Verify consensus (at least 2 sources agree)
  if (results.length < 2) {
    throw new Error('Not enough sources responded');
  }
  
  // Check if results match
  const firstResult = results[0];
  const matchingResults = results.filter(r => 
    JSON.stringify(r.winningNumbers) === JSON.stringify(firstResult.winningNumbers)
  );
  
  if (matchingResults.length >= 2) {
    return firstResult;
  }
  
  throw new Error('Results do not match across sources');
}

export async function submitToBlockchain(roundId: number, result: LotteryResult) {
  // Extract the number we're betting on (e.g., PowerBall or first winning number)
  const numberToUse = result.powerBall || result.winningNumbers[0];
  
  // Submit to smart contract
  const contract = getContract();
  const tx = await contract.finalizeRound(roundId, numberToUse);
  await tx.wait();
  
  console.log(`Round ${roundId} finalized with number ${numberToUse}`);
}
```

---

## 5. Rate Limiting - Prevent Abuse

### Redis-Based Rate Limiting

```typescript
// middleware/rateLimit.ts
import { Redis } from '@upstash/redis';
import { Ratelimit } from '@upstash/ratelimit';

const redis = Redis.fromEnv();

// Create rate limiter
const ratelimit = new Ratelimit({
  redis,
  limiter: Ratelimit.slidingWindow(10, '1 m'), // 10 requests per minute
  analytics: true,
});

// Different limits for different endpoints
const betRateLimit = new Ratelimit({
  redis,
  limiter: Ratelimit.slidingWindow(5, '1 m'), // 5 bets per minute
});

const claimRateLimit = new Ratelimit({
  redis,
  limiter: Ratelimit.slidingWindow(3, '5 m'), // 3 claims per 5 minutes
});

export async function checkRateLimit(
  identifier: string, 
  limiter: Ratelimit = ratelimit
) {
  const { success, limit, reset, remaining } = await limiter.limit(identifier);
  
  return {
    allowed: success,
    limit,
    remaining,
    reset: new Date(reset),
  };
}

// Middleware for Next.js API routes
export function withRateLimit(handler: any, limiter: Ratelimit = ratelimit) {
  return async (req: any, res: any) => {
    // Use IP address or wallet address as identifier
    const identifier = req.headers['x-forwarded-for'] || 
                      req.connection.remoteAddress ||
                      req.body.walletAddress ||
                      'anonymous';
    
    const rateLimitResult = await checkRateLimit(identifier, limiter);
    
    // Set rate limit headers
    res.setHeader('X-RateLimit-Limit', rateLimitResult.limit);
    res.setHeader('X-RateLimit-Remaining', rateLimitResult.remaining);
    res.setHeader('X-RateLimit-Reset', rateLimitResult.reset.getTime());
    
    if (!rateLimitResult.allowed) {
      return res.status(429).json({
        error: 'Too many requests',
        retryAfter: rateLimitResult.reset,
      });
    }
    
    return handler(req, res);
  };
}

// Usage
export default withRateLimit(async function handler(req, res) {
  // Your API logic here
  return res.json({ success: true });
}, betRateLimit);
```

### IP-Based + Wallet-Based Rate Limiting

```typescript
// Advanced rate limiting combining IP and wallet
export async function checkAdvancedRateLimit(
  ip: string,
  walletAddress?: string
) {
  const checks = [];
  
  // Check IP rate limit
  checks.push(checkRateLimit(`ip:${ip}`));
  
  // Check wallet rate limit (if provided)
  if (walletAddress) {
    checks.push(checkRateLimit(`wallet:${walletAddress}`, betRateLimit));
  }
  
  const results = await Promise.all(checks);
  
  // All checks must pass
  const allAllowed = results.every(r => r.allowed);
  const mostRestrictive = results.reduce((min, r) => 
    r.remaining < min.remaining ? r : min
  );
  
  return {
    allowed: allAllowed,
    limit: mostRestrictive.limit,
    remaining: mostRestrictive.remaining,
    reset: mostRestrictive.reset,
  };
}
```

---

## 6. DDoS Protection - Cloudflare Configuration

### Cloudflare Setup

```javascript
// cloudflare-config.js
module.exports = {
  // Security level
  security_level: 'high', // Options: off, essentially_off, low, medium, high, under_attack
  
  // Challenge passage (how long between challenges)
  challenge_ttl: 900, // 15 minutes
  
  // Browser integrity check
  browser_check: 'on',
  
  // Email obfuscation
  email_obfuscation: 'on',
  
  // Server side exclude
  server_side_exclude: 'on',
  
  // TLS settings
  tls_1_3: 'on',
  min_tls_version: '1.2',
  
  // Bot fight mode
  bot_fight_mode: true,
  
  // Rate limiting rules
  rateLimit: {
    rules: [
      {
        id: 'bet-endpoint',
        description: 'Rate limit betting endpoint',
        match: {
          request: {
            url: {
              path: {
                contains: '/api/bet'
              }
            }
          }
        },
        action: {
          mode: 'challenge', // challenge, block, simulate
          timeout: 60, // seconds
          threshold: 10 // requests
        }
      },
      {
        id: 'api-general',
        description: 'General API rate limit',
        match: {
          request: {
            url: {
              path: {
                startsWith: '/api/'
              }
            }
          }
        },
        action: {
          mode: 'challenge',
          timeout: 60,
          threshold: 30
        }
      }
    ]
  },
  
  // Firewall rules
  firewall: {
    rules: [
      {
        id: 'block-suspicious-countries',
        description: 'Block high-risk countries (optional)',
        expression: '(ip.geoip.country in {"CN" "RU"}) and not (ip.src in $whitelist)',
        action: 'challenge'
      },
      {
        id: 'block-known-bots',
        description: 'Block known bad bots',
        expression: '(cf.client.bot) and not (cf.bot_management.verified_bot)',
        action: 'block'
      },
      {
        id: 'challenge-high-request-rate',
        description: 'Challenge users with high request rates',
        expression: '(http.request.uri.path contains "/api/") and (cf.threat_score > 10)',
        action: 'managed_challenge'
      }
    ]
  },
  
  // Page rules
  pageRules: [
    {
      targets: ['/api/*'],
      actions: [
        { id: 'cache_level', value: 'bypass' }, // Don't cache API responses
        { id: 'security_level', value: 'high' },
        { id: 'browser_check', value: 'on' }
      ]
    },
    {
      targets: ['/*.js', '/*.css', '/*.png', '/*.jpg'],
      actions: [
        { id: 'cache_level', value: 'aggressive' },
        { id: 'edge_cache_ttl', value: 86400 } // 24 hours
      ]
    }
  ],
  
  // WAF (Web Application Firewall)
  waf: {
    mode: 'on',
    sensitivity: 'high',
    packages: [
      'OWASP ModSecurity Core Rule Set',
      'Cloudflare Specials',
      'Cloudflare WordPress'
    ]
  }
};
```

### Cloudflare Workers for Additional Protection

```javascript
// cloudflare-worker.js
addEventListener('fetch', event => {
  event.respondWith(handleRequest(event.request));
});

async function handleRequest(request) {
  const url = new URL(request.url);
  
  // Block requests without valid origin
  if (url.pathname.startsWith('/api/')) {
    const origin = request.headers.get('Origin');
    const referer = request.headers.get('Referer');
    
    if (!origin && !referer) {
      return new Response('Forbidden', { status: 403 });
    }
  }
  
  // Check for suspicious user agents
  const userAgent = request.headers.get('User-Agent') || '';
  const suspiciousPatterns = ['bot', 'crawler', 'scraper', 'scanner'];
  
  if (suspiciousPatterns.some(pattern => userAgent.toLowerCase().includes(pattern))) {
    // Challenge suspicious requests
    return new Response('Please verify you are human', {
      status: 403,
      headers: {
        'CF-Challenge': 'true'
      }
    });
  }
  
  // Add custom security headers
  const response = await fetch(request);
  const newHeaders = new Headers(response.headers);
  
  newHeaders.set('X-Content-Type-Options', 'nosniff');
  newHeaders.set('X-Frame-Options', 'DENY');
  newHeaders.set('X-XSS-Protection', '1; mode=block');
  newHeaders.set('Referrer-Policy', 'strict-origin-when-cross-origin');
  newHeaders.set('Permissions-Policy', 'geolocation=(), microphone=(), camera=()');
  
  return new Response(response.body, {
    status: response.status,
    statusText: response.statusText,
    headers: newHeaders
  });
}
```

---

## 7. Wallet Connection Security

### Secure Wallet Integration

```typescript
// hooks/useWalletAuth.ts
import { useState, useEffect } from 'react';
import { ethers } from 'ethers';

export function useWalletAuth() {
  const [account, setAccount] = useState<string | null>(null);
  const [isAuthenticated, setIsAuthenticated] = useState(false);
  
  const connectWallet = async () => {
    if (!window.ethereum) {
      throw new Error('Please install MetaMask');
    }
    
    try {
      // Request accounts
      const provider = new ethers.providers.Web3Provider(window.ethereum);
      await provider.send("eth_requestAccounts", []);
      
      const signer = provider.getSigner();
      const address = await signer.getAddress();
      
      // Verify network
      const network = await provider.getNetwork();
      if (network.chainId !== 137) { // Polygon mainnet
        throw new Error('Please switch to Polygon network');
      }
      
      // Generate nonce for signing
      const nonce = Date.now().toString();
      const message = `Sign this message to authenticate with LottoChain.\n\nNonce: ${nonce}\nAddress: ${address}`;
      
      // Request signature
      const signature = await signer.signMessage(message);
      
      // Verify signature on backend
      const verified = await verifySignature(address, message, signature);
      
      if (verified) {
        setAccount(address);
        setIsAuthenticated(true);
        
        // Store session
        localStorage.setItem('wallet', address);
        localStorage.setItem('signature', signature);
        
        return address;
      }
      
      throw new Error('Signature verification failed');
      
    } catch (error) {
      console.error('Wallet connection error:', error);
      throw error;
    }
  };
  
  const disconnectWallet = () => {
    setAccount(null);
    setIsAuthenticated(false);
    localStorage.removeItem('wallet');
    localStorage.removeItem('signature');
  };
  
  // Listen for account changes
  useEffect(() => {
    if (window.ethereum) {
      window.ethereum.on('accountsChanged', (accounts: string[]) => {
        if (accounts.length === 0) {
          disconnectWallet();
        } else if (accounts[0] !== account) {
          // Account changed, re-authenticate
          connectWallet();
        }
      });
      
      window.ethereum.on('chainChanged', () => {
        // Reload page on chain change
        window.location.reload();
      });
    }
    
    return () => {
      if (window.ethereum) {
        window.ethereum.removeAllListeners('accountsChanged');
        window.ethereum.removeAllListeners('chainChanged');
      }
    };
  }, [account]);
  
  return {
    account,
    isAuthenticated,
    connectWallet,
    disconnectWallet
  };
}

// Backend signature verification
async function verifySignature(
  address: string,
  message: string,
  signature: string
): Promise<boolean> {
  try {
    const response = await fetch('/api/auth/verify', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ address, message, signature })
    });
    
    const data = await response.json();
    return data.verified === true;
  } catch (error) {
    console.error('Verification error:', error);
    return false;
  }
}
```

### Backend Signature Verification

```typescript
// api/auth/verify.ts
import { ethers } from 'ethers';

export async function POST(req: Request) {
  const { address, message, signature } = await req.json();
  
  try {
    // Recover signer from signature
    const recoveredAddress = ethers.utils.verifyMessage(message, signature);
    
    // Check if recovered address matches claimed address
    if (recoveredAddress.toLowerCase() !== address.toLowerCase()) {
      return Response.json({ verified: false }, { status: 401 });
    }
    
    // Check nonce is recent (within 5 minutes)
    const nonceMatch = message.match(/Nonce: (\d+)/);
    if (nonceMatch) {
      const nonce = parseInt(nonceMatch[1]);
      const age = Date.now() - nonce;
      
      if (age > 5 * 60 * 1000) {
        return Response.json({ 
          verified: false, 
          error: 'Nonce expired' 
        }, { status: 401 });
      }
    }
    
    // Generate session token
    const sessionToken = generateSessionToken(address);
    
    return Response.json({ 
      verified: true,
      sessionToken
    });
    
  } catch (error) {
    return Response.json({ 
      verified: false,
      error: 'Invalid signature'
    }, { status: 401 });
  }
}

function generateSessionToken(address: string): string {
  // Generate JWT or session token
  const token = ethers.utils.id(`${address}-${Date.now()}-${Math.random()}`);
  return token;
}
```

---

## 8. Data Encryption - Protect Sensitive Information

### Encrypt User Data in Redis

```typescript
// utils/encryption.ts
import crypto from 'crypto';

const ENCRYPTION_KEY = process.env.ENCRYPTION_KEY!; // Must be 32 bytes
const ALGORITHM = 'aes-256-gcm';

export function encrypt(text: string): string {
  // Generate random IV
  const iv = crypto.randomBytes(16);
  
  // Create cipher
  const cipher = crypto.createCipheriv(
    ALGORITHM,
    Buffer.from(ENCRYPTION_KEY, 'hex'),
    iv
  );
  
  // Encrypt
  let encrypted = cipher.update(text, 'utf8', 'hex');
  encrypted += cipher.final('hex');
  
  // Get auth tag
  const authTag = cipher.getAuthTag();
  
  // Return IV + authTag + encrypted
  return iv.toString('hex') + ':' + authTag.toString('hex') + ':' + encrypted;
}

export function decrypt(encryptedData: string): string {
  // Split into IV, authTag, and encrypted text
  const [ivHex, authTagHex, encrypted] = encryptedData.split(':');
  
  const iv = Buffer.from(ivHex, 'hex');
  const authTag = Buffer.from(authTagHex, 'hex');
  
  // Create decipher
  const decipher = crypto.createDecipheriv(
    ALGORITHM,
    Buffer.from(ENCRYPTION_KEY, 'hex'),
    iv
  );
  
  decipher.setAuthTag(authTag);
  
  // Decrypt
  let decrypted = decipher.update(encrypted, 'hex', 'utf8');
  decrypted += decipher.final('utf8');
  
  return decrypted;
}

// Usage in Redis
import { Redis } from '@upstash/redis';

const redis = Redis.fromEnv();

export async function storeUserData(userId: string, data: any) {
  const encrypted = encrypt(JSON.stringify(data));
  await redis.set(`user:${userId}`, encrypted);
}

export async function getUserData(userId: string): Promise<any> {
  const encrypted = await redis.get(`user:${userId}`);
  if (!encrypted) return null;
  
  const decrypted = decrypt(encrypted as string);
  return JSON.parse(decrypted);
}
```

### Encrypt Environment Variables

```bash
# Generate encryption key
openssl rand -hex 32

# .env.local (never commit!)
ENCRYPTION_KEY=your-32-byte-hex-key-here
REDIS_URL=your-redis-url
PRIVATE_KEY=your-wallet-private-key
```

```typescript
// Use encrypted env vars in production
import { config } from 'dotenv';

// Load and validate
config();

const requiredEnvVars = [
  'ENCRYPTION_KEY',
  'REDIS_URL',
  'PRIVATE_KEY',
  'CONTRACT_ADDRESS'
];

for (const key of requiredEnvVars) {
  if (!process.env[key]) {
    throw new Error(`Missing required environment variable: ${key}`);
  }
}

// Encrypt private key in memory (additional security)
const encryptedPrivateKey = encrypt(process.env.PRIVATE_KEY!);
```

---

## 9-12. Additional Security Measures

### 9. Audit Logging

```typescript
// utils/auditLog.ts
import { Redis } from '@upstash/redis';

const redis = Redis.fromEnv();

export enum AuditAction {
  BET_PLACED = 'BET_PLACED',
  WINNINGS_CLAIMED = 'WINNINGS_CLAIMED',
  REFUND_CLAIMED = 'REFUND_CLAIMED',
  ROUND_CREATED = 'ROUND_CREATED',
  ROUND_FINALIZED = 'ROUND_FINALIZED',
  ADMIN_ACTION = 'ADMIN_ACTION'
}

export async function logAudit(
  action: AuditAction,
  userId: string,
  details: any,
  req?: any
) {
  const logEntry = {
    action,
    userId,
    details,
    timestamp: new Date().toISOString(),
    ip: req?.ip || req?.headers?.['x-forwarded-for'],
    userAgent: req?.headers?.['user-agent'],
    sessionId: req?.sessionId
  };
  
  // Store in Redis list
  await redis.lpush('audit:logs', JSON.stringify(logEntry));
  
  // Also log to console/file
  console.log('[AUDIT]', logEntry);
  
  // Keep only last 10000 entries
  await redis.ltrim('audit:logs', 0, 9999);
}
```

### 10. Error Handling

```typescript
// middleware/errorHandler.ts
export function errorHandler(err: any, req: any, res: any, next: any) {
  // Log error internally
  console.error('[ERROR]', {
    message: err.message,
    stack: err.stack,
    url: req.url,
    method: req.method,
    userId: req.userId
  });
  
  // Don't expose internal errors to client
  const isDev = process.env.NODE_ENV === 'development';
  
  res.status(err.status || 500).json({
    error: isDev ? err.message : 'An error occurred',
    code: err.code || 'INTERNAL_ERROR',
    ...(isDev && { stack: err.stack })
  });
}
```

### 11. CORS Configuration

```typescript
// middleware/cors.ts
import cors from 'cors';

const allowedOrigins = [
  'https://yourdomain.com',
  'https://www.yourdomain.com',
  ...(process.env.NODE_ENV === 'development' ? ['http://localhost:3000'] : [])
];

export const corsMiddleware = cors({
  origin: (origin, callback) => {
    if (!origin || allowedOrigins.includes(origin)) {
      callback(null, true);
    } else {
      callback(new Error('Not allowed by CORS'));
    }
  },
  credentials: true,
  methods: ['GET', 'POST', 'PUT', 'DELETE'],
  allowedHeaders: ['Content-Type', 'Authorization', 'X-Session-Token'],
  exposedHeaders: ['X-RateLimit-Limit', 'X-RateLimit-Remaining'],
  maxAge: 86400 // 24 hours
});
```

### 12. Environment Security

```typescript
// config/validateEnv.ts
const requiredEnvVars = [
  'NEXT_PUBLIC_CONTRACT_ADDRESS',
  'NEXT_PUBLIC_CHAIN_ID',
  'PRIVATE_KEY',
  'ENCRYPTION_KEY',
  'REDIS_URL',
  'CHAINLINK_ORACLE_ADDRESS'
];

const optionalEnvVars = [
  'SENTRY_DSN',
  'ANALYTICS_ID'
];

export function validateEnvironment() {
  const missing = requiredEnvVars.filter(key => !process.env[key]);
  
  if (missing.length > 0) {
    throw new Error(
      `Missing required environment variables: ${missing.join(', ')}`
    );
  }
  
  // Validate format
  if (process.env.PRIVATE_KEY && !process.env.PRIVATE_KEY.startsWith('0x')) {
    throw new Error('PRIVATE_KEY must start with 0x');
  }
  
  if (process.env.ENCRYPTION_KEY && process.env.ENCRYPTION_KEY.length !== 64) {
    throw new Error('ENCRYPTION_KEY must be 32 bytes (64 hex characters)');
  }
  
  console.log('✅ Environment variables validated');
}

// Run on startup
validateEnvironment();
```

## Security Checklist

Before deployment, ensure all of these are implemented:

- [ ] Smart contract audited (self-audit + professional audit)
- [ ] Reentrancy protection on all withdrawal functions
- [ ] Access control with role-based permissions
- [ ] Input validation on all user inputs (frontend + backend + smart contract)
- [ ] Multiple oracle sources with consensus mechanism
- [ ] Rate limiting on all API endpoints
- [ ] DDoS protection via Cloudflare
- [ ] Secure wallet connection with signature verification
- [ ] Data encryption for sensitive information
- [ ] Comprehensive audit logging
- [ ] Proper error handling (don't expose internals)
- [ ] Strict CORS policy
- [ ] Environment variables secured and validated
- [ ] HTTPS everywhere (SSL/TLS)
- [ ] Regular security updates for dependencies
- [ ] Backup and disaster recovery plan
- [ ] Monitoring and alerting system
- [ ] Bug bounty program (post-launch)

## Testing Security

```bash
# Run security tests
npm run test:security

# Check for vulnerabilities in dependencies
npm audit
npm audit fix

# Run static analysis
npm run lint:security

# Test smart contract with Mythril or Slither
mythril analyze contracts/P2PLottery.sol
slither contracts/P2PLottery.sol

# Penetration testing (use tools like OWASP ZAP)
```

## Incident Response Plan

1. **Detection**: Monitor logs and alerts
2. **Containment**: Pause contract if critical bug found
3. **Investigation**: Analyze the issue
4. **Resolution**: Fix the vulnerability
5. **Recovery**: Resume operations
6. **Post-mortem**: Document and learn

---

This security guide provides comprehensive protection for your P2P lottery platform. Implement all 12 techniques for maximum security! 🔒
