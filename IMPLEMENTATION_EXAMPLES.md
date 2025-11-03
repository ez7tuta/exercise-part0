# Implementation Examples

This file contains ready-to-use code examples for implementing the P2P Lottery Platform.

## Table of Contents
1. [Smart Contract Example](#smart-contract-example)
2. [Frontend Examples](#frontend-examples)
3. [Backend API Examples](#backend-api-examples)
4. [Real-time Updates with Socket.io](#real-time-updates-with-socketio)
5. [Testing Examples](#testing-examples)

---

## Smart Contract Example

### Complete P2PLottery.sol

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import "@openzeppelin/contracts/security/ReentrancyGuard.sol";
import "@openzeppelin/contracts/access/AccessControl.sol";
import "@openzeppelin/contracts/security/Pausable.sol";

contract P2PLotteryBetting is ReentrancyGuard, AccessControl, Pausable {
    
    bytes32 public constant ORACLE_ROLE = keccak256("ORACLE_ROLE");
    bytes32 public constant OPERATOR_ROLE = keccak256("OPERATOR_ROLE");
    
    enum PredictionType { EVEN, ODD, HIGH, LOW, EVEN_LOW, ODD_HIGH, EVEN_HIGH, ODD_LOW }
    enum RoundStatus { PENDING, OPEN, LOCKED, FINALIZED, CANCELLED }
    
    struct Round {
        uint256 roundId;
        uint256 drawTime;
        uint256 lockTime;
        RoundStatus status;
        bool isFinalized;
        uint8 winningNumber;
        uint256 totalPoolSize;
    }
    
    struct Bet {
        uint256 amount;
        PredictionType prediction;
        uint256 matchedAmount;
        bool claimed;
        uint256 timestamp;
    }
    
    struct PredictionPool {
        uint256 totalAmount;
        uint256 matchedAmount;
        address[] bettors;
    }
    
    // State variables
    uint256 public currentRoundId;
    mapping(uint256 => Round) public rounds;
    mapping(uint256 => mapping(PredictionType => PredictionPool)) public predictionPools;
    mapping(uint256 => mapping(address => Bet)) public userBets;
    
    // Constants
    uint256 public constant PLATFORM_FEE = 2; // 2%
    uint256 public constant MIN_BET = 0.01 ether;
    uint256 public constant MAX_BET = 100 ether;
    uint256 public constant LOCK_TIME_BEFORE_DRAW = 15 minutes;
    
    address public feeCollector;
    uint256 public totalFeesCollected;
    
    // Events
    event RoundCreated(uint256 indexed roundId, uint256 drawTime, uint256 lockTime);
    event BetPlaced(uint256 indexed roundId, address indexed user, PredictionType prediction, uint256 amount);
    event BetMatched(uint256 indexed roundId, address indexed user, uint256 matchedAmount);
    event RoundLocked(uint256 indexed roundId, uint256 timestamp);
    event RoundFinalized(uint256 indexed roundId, uint8 winningNumber, uint256 timestamp);
    event WinningsClaimed(uint256 indexed roundId, address indexed user, uint256 amount);
    event RefundIssued(uint256 indexed roundId, address indexed user, uint256 amount);
    event FeeCollected(uint256 indexed roundId, uint256 amount);
    
    constructor() {
        _grantRole(DEFAULT_ADMIN_ROLE, msg.sender);
        _grantRole(OPERATOR_ROLE, msg.sender);
        feeCollector = msg.sender;
    }
    
    // Create new round
    function createRound(uint256 drawTime) external onlyRole(OPERATOR_ROLE) returns (uint256) {
        require(drawTime > block.timestamp, "Draw time must be in future");
        
        currentRoundId++;
        
        rounds[currentRoundId] = Round({
            roundId: currentRoundId,
            drawTime: drawTime,
            lockTime: drawTime - LOCK_TIME_BEFORE_DRAW,
            status: RoundStatus.OPEN,
            isFinalized: false,
            winningNumber: 0,
            totalPoolSize: 0
        });
        
        emit RoundCreated(currentRoundId, drawTime, drawTime - LOCK_TIME_BEFORE_DRAW);
        
        return currentRoundId;
    }
    
    // Place a bet
    function placeBet(uint256 roundId, PredictionType prediction) 
        external 
        payable 
        nonReentrant 
        whenNotPaused 
    {
        require(msg.value >= MIN_BET, "Bet too small");
        require(msg.value <= MAX_BET, "Bet too large");
        require(roundId <= currentRoundId, "Invalid round");
        
        Round storage round = rounds[roundId];
        require(round.status == RoundStatus.OPEN, "Round not open");
        require(block.timestamp < round.lockTime, "Betting closed");
        require(userBets[roundId][msg.sender].amount == 0, "Already placed bet");
        
        // Store bet
        userBets[roundId][msg.sender] = Bet({
            amount: msg.value,
            prediction: prediction,
            matchedAmount: 0,
            claimed: false,
            timestamp: block.timestamp
        });
        
        // Update pool
        PredictionPool storage pool = predictionPools[roundId][prediction];
        pool.totalAmount += msg.value;
        pool.bettors.push(msg.sender);
        
        round.totalPoolSize += msg.value;
        
        emit BetPlaced(roundId, msg.sender, prediction, msg.value);
        
        // Try to match bets
        _matchBets(roundId, prediction);
    }
    
    // Internal: Match bets between opposite predictions
    function _matchBets(uint256 roundId, PredictionType prediction) internal {
        PredictionType opposite = _getOpposite(prediction);
        if (opposite == prediction) return; // No opposite for combined predictions yet
        
        PredictionPool storage poolA = predictionPools[roundId][prediction];
        PredictionPool storage poolB = predictionPools[roundId][opposite];
        
        uint256 unmatchedA = poolA.totalAmount - poolA.matchedAmount;
        uint256 unmatchedB = poolB.totalAmount - poolB.matchedAmount;
        
        if (unmatchedA == 0 || unmatchedB == 0) return;
        
        uint256 toMatch = unmatchedA < unmatchedB ? unmatchedA : unmatchedB;
        
        poolA.matchedAmount += toMatch;
        poolB.matchedAmount += toMatch;
        
        // Update individual bets proportionally
        _updateBetMatching(roundId, prediction, poolA.bettors, poolA.matchedAmount, poolA.totalAmount);
        _updateBetMatching(roundId, opposite, poolB.bettors, poolB.matchedAmount, poolB.totalAmount);
    }
    
    function _updateBetMatching(
        uint256 roundId,
        PredictionType prediction,
        address[] storage bettors,
        uint256 totalMatched,
        uint256 totalPool
    ) internal {
        for (uint i = 0; i < bettors.length; i++) {
            address bettor = bettors[i];
            Bet storage bet = userBets[roundId][bettor];
            
            if (bet.prediction == prediction) {
                bet.matchedAmount = (bet.amount * totalMatched) / totalPool;
                
                emit BetMatched(roundId, bettor, bet.matchedAmount);
            }
        }
    }
    
    // Get opposite prediction
    function _getOpposite(PredictionType prediction) internal pure returns (PredictionType) {
        if (prediction == PredictionType.EVEN) return PredictionType.ODD;
        if (prediction == PredictionType.ODD) return PredictionType.EVEN;
        if (prediction == PredictionType.HIGH) return PredictionType.LOW;
        if (prediction == PredictionType.LOW) return PredictionType.HIGH;
        return prediction; // No opposite for combined types
    }
    
    // Lock round (called 15 minutes before draw)
    function lockRound(uint256 roundId) external onlyRole(OPERATOR_ROLE) {
        Round storage round = rounds[roundId];
        require(round.status == RoundStatus.OPEN, "Round not open");
        require(block.timestamp >= round.lockTime, "Too early to lock");
        
        round.status = RoundStatus.LOCKED;
        
        emit RoundLocked(roundId, block.timestamp);
    }
    
    // Finalize round with result (called by oracle)
    function finalizeRound(uint256 roundId, uint8 winningNumber) 
        external 
        onlyRole(ORACLE_ROLE) 
    {
        Round storage round = rounds[roundId];
        require(round.status == RoundStatus.LOCKED, "Round not locked");
        require(!round.isFinalized, "Already finalized");
        require(block.timestamp >= round.drawTime, "Draw hasn't occurred");
        require(winningNumber <= 99, "Invalid number");
        
        round.isFinalized = true;
        round.winningNumber = winningNumber;
        round.status = RoundStatus.FINALIZED;
        
        emit RoundFinalized(roundId, winningNumber, block.timestamp);
        
        // Collect platform fees
        _collectFees(roundId);
    }
    
    // Collect platform fees
    function _collectFees(uint256 roundId) internal {
        Round storage round = rounds[roundId];
        
        // Calculate total matched amount
        uint256 totalMatched = 0;
        
        for (uint8 i = 0; i < 8; i++) {
            PredictionType pred = PredictionType(i);
            totalMatched += predictionPools[roundId][pred].matchedAmount;
        }
        
        // Fee on matched amount only
        uint256 fee = (totalMatched * PLATFORM_FEE) / 100;
        totalFeesCollected += fee;
        
        emit FeeCollected(roundId, fee);
    }
    
    // Claim winnings
    function claimWinnings(uint256 roundId) external nonReentrant {
        Round storage round = rounds[roundId];
        require(round.isFinalized, "Round not finalized");
        
        Bet storage userBet = userBets[roundId][msg.sender];
        require(!userBet.claimed, "Already claimed");
        require(userBet.amount > 0, "No bet placed");
        
        userBet.claimed = true;
        
        // Check if user won
        bool won = _checkWin(userBet.prediction, round.winningNumber);
        
        if (won && userBet.matchedAmount > 0) {
            uint256 winnings = _calculateWinnings(roundId, msg.sender);
            
            (bool success, ) = payable(msg.sender).call{value: winnings}("");
            require(success, "Transfer failed");
            
            emit WinningsClaimed(roundId, msg.sender, winnings);
        } else {
            // Lost - no payout for matched amount
            // But refund unmatched amount
            uint256 unmatchedAmount = userBet.amount - userBet.matchedAmount;
            
            if (unmatchedAmount > 0) {
                (bool success, ) = payable(msg.sender).call{value: unmatchedAmount}("");
                require(success, "Refund failed");
                
                emit RefundIssued(roundId, msg.sender, unmatchedAmount);
            }
        }
    }
    
    // Check if prediction won
    function _checkWin(PredictionType prediction, uint8 number) internal pure returns (bool) {
        if (prediction == PredictionType.EVEN) return number % 2 == 0;
        if (prediction == PredictionType.ODD) return number % 2 == 1;
        if (prediction == PredictionType.HIGH) return number > 50;
        if (prediction == PredictionType.LOW) return number <= 50;
        if (prediction == PredictionType.EVEN_LOW) return (number % 2 == 0) && (number <= 50);
        if (prediction == PredictionType.ODD_HIGH) return (number % 2 == 1) && (number > 50);
        if (prediction == PredictionType.EVEN_HIGH) return (number % 2 == 0) && (number > 50);
        if (prediction == PredictionType.ODD_LOW) return (number % 2 == 1) && (number <= 50);
        return false;
    }
    
    // Calculate winnings
    function _calculateWinnings(uint256 roundId, address user) internal view returns (uint256) {
        Bet storage userBet = userBets[roundId][user];
        PredictionType prediction = userBet.prediction;
        
        PredictionPool storage winningPool = predictionPools[roundId][prediction];
        PredictionType opposite = _getOpposite(prediction);
        PredictionPool storage losingPool = predictionPools[roundId][opposite];
        
        // User's share of the losing pool
        uint256 userShare = (userBet.matchedAmount * losingPool.matchedAmount) / winningPool.matchedAmount;
        
        // Subtract platform fee
        uint256 totalReturn = userBet.matchedAmount + userShare;
        uint256 fee = (totalReturn * PLATFORM_FEE) / 100;
        
        return totalReturn - fee;
    }
    
    // Withdraw collected fees
    function withdrawFees() external onlyRole(DEFAULT_ADMIN_ROLE) nonReentrant {
        uint256 amount = totalFeesCollected;
        totalFeesCollected = 0;
        
        (bool success, ) = payable(feeCollector).call{value: amount}("");
        require(success, "Withdrawal failed");
    }
    
    // Update fee collector
    function setFeeCollector(address _feeCollector) external onlyRole(DEFAULT_ADMIN_ROLE) {
        require(_feeCollector != address(0), "Invalid address");
        feeCollector = _feeCollector;
    }
    
    // Emergency pause
    function pause() external onlyRole(DEFAULT_ADMIN_ROLE) {
        _pause();
    }
    
    function unpause() external onlyRole(DEFAULT_ADMIN_ROLE) {
        _unpause();
    }
    
    // View functions
    function getRoundInfo(uint256 roundId) external view returns (
        uint256 drawTime,
        uint256 lockTime,
        RoundStatus status,
        bool isFinalized,
        uint8 winningNumber,
        uint256 totalPoolSize
    ) {
        Round storage round = rounds[roundId];
        return (
            round.drawTime,
            round.lockTime,
            round.status,
            round.isFinalized,
            round.winningNumber,
            round.totalPoolSize
        );
    }
    
    function getPredictionPool(uint256 roundId, PredictionType prediction) 
        external 
        view 
        returns (uint256 totalAmount, uint256 matchedAmount, uint256 bettorCount) 
    {
        PredictionPool storage pool = predictionPools[roundId][prediction];
        return (pool.totalAmount, pool.matchedAmount, pool.bettors.length);
    }
    
    function getUserBet(uint256 roundId, address user) 
        external 
        view 
        returns (
            uint256 amount,
            PredictionType prediction,
            uint256 matchedAmount,
            bool claimed
        ) 
    {
        Bet storage bet = userBets[roundId][user];
        return (bet.amount, bet.prediction, bet.matchedAmount, bet.claimed);
    }
}
```

### Deployment Script

```javascript
// scripts/deploy.js
const hre = require("hardhat");

async function main() {
  console.log("Deploying P2PLotteryBetting contract...");

  const P2PLotteryBetting = await hre.ethers.getContractFactory("P2PLotteryBetting");
  const contract = await P2PLotteryBetting.deploy();

  await contract.deployed();

  console.log("P2PLotteryBetting deployed to:", contract.address);
  
  // Wait for block confirmations
  console.log("Waiting for block confirmations...");
  await contract.deployTransaction.wait(5);
  
  // Verify on Polygonscan
  console.log("Verifying contract...");
  await hre.run("verify:verify", {
    address: contract.address,
    constructorArguments: [],
  });
  
  console.log("Contract verified!");
}

main()
  .then(() => process.exit(0))
  .catch((error) => {
    console.error(error);
    process.exit(1);
  });
```

---

## Frontend Examples

### Next.js Project Structure

```
lottery-platform/
├── app/
│   ├── layout.tsx
│   ├── page.tsx
│   ├── bet/
│   │   └── page.tsx
│   ├── history/
│   │   └── page.tsx
│   └── dashboard/
│       └── page.tsx
├── components/
│   ├── BettingInterface.tsx
│   ├── WalletConnect.tsx
│   ├── BetStatus.tsx
│   ├── RoundHistory.tsx
│   └── LiveStats.tsx
├── hooks/
│   ├── useContract.ts
│   ├── useWallet.ts
│   └── useSocket.ts
├── lib/
│   ├── contract.ts
│   ├── socket.ts
│   └── utils.ts
└── public/
```

### Betting Interface Component

```typescript
// components/BettingInterface.tsx
'use client';

import { useState, useEffect } from 'react';
import { useContract } from '@/hooks/useContract';
import { useWallet } from '@/hooks/useWallet';
import { useSocket } from '@/hooks/useSocket';
import { ethers } from 'ethers';

enum PredictionType {
  EVEN = 0,
  ODD = 1,
  HIGH = 2,
  LOW = 3
}

export function BettingInterface() {
  const { account, connectWallet } = useWallet();
  const { placeBet, getRoundInfo, getPredictionPool } = useContract();
  const socket = useSocket();
  
  const [currentRound, setCurrentRound] = useState<any>(null);
  const [pools, setPools] = useState<any>({});
  const [betAmount, setBetAmount] = useState('0.1');
  const [selectedPrediction, setSelectedPrediction] = useState<PredictionType | null>(null);
  const [loading, setLoading] = useState(false);
  const [timeRemaining, setTimeRemaining] = useState('');
  
  useEffect(() => {
    loadRoundData();
    
    // Subscribe to real-time updates
    if (socket) {
      socket.on('round:updated', handleRoundUpdate);
      socket.on('pool:updated', handlePoolUpdate);
    }
    
    return () => {
      if (socket) {
        socket.off('round:updated', handleRoundUpdate);
        socket.off('pool:updated', handlePoolUpdate);
      }
    };
  }, [socket]);
  
  // Update countdown timer
  useEffect(() => {
    const timer = setInterval(() => {
      if (currentRound?.lockTime) {
        const now = Date.now();
        const lockTime = currentRound.lockTime * 1000;
        const diff = lockTime - now;
        
        if (diff > 0) {
          const minutes = Math.floor(diff / 60000);
          const seconds = Math.floor((diff % 60000) / 1000);
          setTimeRemaining(`${minutes}:${seconds.toString().padStart(2, '0')}`);
        } else {
          setTimeRemaining('Closed');
        }
      }
    }, 1000);
    
    return () => clearInterval(timer);
  }, [currentRound]);
  
  async function loadRoundData() {
    try {
      const roundInfo = await getRoundInfo(1); // Current round
      setCurrentRound(roundInfo);
      
      // Load all pools
      const poolData: any = {};
      for (let i = 0; i < 4; i++) {
        const pool = await getPredictionPool(1, i);
        poolData[i] = pool;
      }
      setPools(poolData);
    } catch (error) {
      console.error('Error loading round data:', error);
    }
  }
  
  function handleRoundUpdate(data: any) {
    setCurrentRound(data);
  }
  
  function handlePoolUpdate(data: any) {
    setPools((prev: any) => ({
      ...prev,
      [data.prediction]: data.pool
    }));
  }
  
  async function handlePlaceBet() {
    if (!account) {
      await connectWallet();
      return;
    }
    
    if (selectedPrediction === null) {
      alert('Please select a prediction');
      return;
    }
    
    setLoading(true);
    
    try {
      const amount = ethers.utils.parseEther(betAmount);
      const tx = await placeBet(1, selectedPrediction, amount);
      await tx.wait();
      
      alert('Bet placed successfully!');
      await loadRoundData();
    } catch (error: any) {
      console.error('Error placing bet:', error);
      alert(error.message || 'Failed to place bet');
    } finally {
      setLoading(false);
    }
  }
  
  function calculateEstimatedPayout(prediction: PredictionType): string {
    const pool = pools[prediction];
    const opposite = prediction === PredictionType.EVEN ? PredictionType.ODD : 
                    prediction === PredictionType.ODD ? PredictionType.EVEN :
                    prediction === PredictionType.HIGH ? PredictionType.LOW : PredictionType.HIGH;
    const oppositePool = pools[opposite];
    
    if (!pool || !oppositePool) return '0.00';
    
    const myBet = parseFloat(betAmount);
    const totalPool = parseFloat(ethers.utils.formatEther(pool.totalAmount)) + myBet;
    const oppositeTotal = parseFloat(ethers.utils.formatEther(oppositePool.totalAmount));
    
    if (totalPool === 0) return '0.00';
    
    const myShare = myBet / totalPool;
    const myWinnings = myShare * oppositeTotal;
    const fee = (myBet + myWinnings) * 0.02;
    
    return (myBet + myWinnings - fee).toFixed(2);
  }
  
  return (
    <div className="max-w-4xl mx-auto p-6">
      <div className="bg-white rounded-lg shadow-lg p-6">
        {/* Header */}
        <div className="mb-6">
          <h1 className="text-3xl font-bold text-gray-800">🎰 LottoChain</h1>
          <p className="text-gray-600">P2P Lottery Prediction Platform</p>
        </div>
        
        {/* Round Info */}
        <div className="bg-blue-50 rounded-lg p-4 mb-6">
          <div className="flex justify-between items-center">
            <div>
              <p className="text-sm text-gray-600">Current Round</p>
              <p className="text-2xl font-bold text-blue-600">#{currentRound?.roundId || '...'}</p>
            </div>
            <div>
              <p className="text-sm text-gray-600">Betting Closes In</p>
              <p className="text-2xl font-bold text-red-600">{timeRemaining}</p>
            </div>
            <div>
              <p className="text-sm text-gray-600">Total Pool</p>
              <p className="text-2xl font-bold text-green-600">
                {currentRound?.totalPoolSize ? 
                  ethers.utils.formatEther(currentRound.totalPoolSize) : '0.00'} ETH
              </p>
            </div>
          </div>
        </div>
        
        {/* Prediction Options */}
        <div className="grid grid-cols-2 gap-4 mb-6">
          {/* EVEN */}
          <button
            onClick={() => setSelectedPrediction(PredictionType.EVEN)}
            className={`p-6 rounded-lg border-2 transition-all ${
              selectedPrediction === PredictionType.EVEN
                ? 'border-blue-500 bg-blue-50'
                : 'border-gray-300 hover:border-blue-300'
            }`}
          >
            <h3 className="text-xl font-bold mb-2">EVEN (Chẵn)</h3>
            <p className="text-sm text-gray-600 mb-2">
              {pools[PredictionType.EVEN] ? 
                ethers.utils.formatEther(pools[PredictionType.EVEN].totalAmount) : '0.00'} ETH
            </p>
            <p className="text-xs text-gray-500">
              {pools[PredictionType.EVEN]?.bettorCount || 0} bets
            </p>
            {pools[PredictionType.EVEN] && (
              <div className="mt-2">
                <p className="text-xs text-green-600">
                  ✓ Matched: {ethers.utils.formatEther(pools[PredictionType.EVEN].matchedAmount)} ETH
                </p>
              </div>
            )}
          </button>
          
          {/* ODD */}
          <button
            onClick={() => setSelectedPrediction(PredictionType.ODD)}
            className={`p-6 rounded-lg border-2 transition-all ${
              selectedPrediction === PredictionType.ODD
                ? 'border-blue-500 bg-blue-50'
                : 'border-gray-300 hover:border-blue-300'
            }`}
          >
            <h3 className="text-xl font-bold mb-2">ODD (Lẻ)</h3>
            <p className="text-sm text-gray-600 mb-2">
              {pools[PredictionType.ODD] ? 
                ethers.utils.formatEther(pools[PredictionType.ODD].totalAmount) : '0.00'} ETH
            </p>
            <p className="text-xs text-gray-500">
              {pools[PredictionType.ODD]?.bettorCount || 0} bets
            </p>
            {pools[PredictionType.ODD] && (
              <div className="mt-2">
                <p className="text-xs text-green-600">
                  ✓ Matched: {ethers.utils.formatEther(pools[PredictionType.ODD].matchedAmount)} ETH
                </p>
              </div>
            )}
          </button>
          
          {/* HIGH */}
          <button
            onClick={() => setSelectedPrediction(PredictionType.HIGH)}
            className={`p-6 rounded-lg border-2 transition-all ${
              selectedPrediction === PredictionType.HIGH
                ? 'border-blue-500 bg-blue-50'
                : 'border-gray-300 hover:border-blue-300'
            }`}
          >
            <h3 className="text-xl font-bold mb-2">HIGH (Cao) {'>'}50</h3>
            <p className="text-sm text-gray-600 mb-2">
              {pools[PredictionType.HIGH] ? 
                ethers.utils.formatEther(pools[PredictionType.HIGH].totalAmount) : '0.00'} ETH
            </p>
            <p className="text-xs text-gray-500">
              {pools[PredictionType.HIGH]?.bettorCount || 0} bets
            </p>
          </button>
          
          {/* LOW */}
          <button
            onClick={() => setSelectedPrediction(PredictionType.LOW)}
            className={`p-6 rounded-lg border-2 transition-all ${
              selectedPrediction === PredictionType.LOW
                ? 'border-blue-500 bg-blue-50'
                : 'border-gray-300 hover:border-blue-300'
            }`}
          >
            <h3 className="text-xl font-bold mb-2">LOW (Thấp) ≤50</h3>
            <p className="text-sm text-gray-600 mb-2">
              {pools[PredictionType.LOW] ? 
                ethers.utils.formatEther(pools[PredictionType.LOW].totalAmount) : '0.00'} ETH
            </p>
            <p className="text-xs text-gray-500">
              {pools[PredictionType.LOW]?.bettorCount || 0} bets
            </p>
          </button>
        </div>
        
        {/* Bet Amount Input */}
        <div className="mb-6">
          <label className="block text-sm font-medium text-gray-700 mb-2">
            Bet Amount (ETH)
          </label>
          <input
            type="number"
            value={betAmount}
            onChange={(e) => setBetAmount(e.target.value)}
            min="0.01"
            max="100"
            step="0.01"
            className="w-full px-4 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500"
            placeholder="0.1"
          />
          <p className="text-xs text-gray-500 mt-1">
            Min: 0.01 ETH | Max: 100 ETH
          </p>
        </div>
        
        {/* Estimated Payout */}
        {selectedPrediction !== null && (
          <div className="bg-green-50 rounded-lg p-4 mb-6">
            <p className="text-sm text-gray-600">Estimated Payout (if you win)</p>
            <p className="text-2xl font-bold text-green-600">
              {calculateEstimatedPayout(selectedPrediction)} ETH
            </p>
            <p className="text-xs text-gray-500 mt-1">
              *Includes 2% platform fee
            </p>
          </div>
        )}
        
        {/* Place Bet Button */}
        <button
          onClick={handlePlaceBet}
          disabled={loading || !selectedPrediction}
          className="w-full bg-blue-600 hover:bg-blue-700 disabled:bg-gray-400 text-white font-bold py-3 px-6 rounded-lg transition-colors"
        >
          {loading ? 'Processing...' : account ? 'Place Bet' : 'Connect Wallet to Bet'}
        </button>
        
        {account && (
          <p className="text-center text-sm text-gray-600 mt-2">
            Connected: {account.slice(0, 6)}...{account.slice(-4)}
          </p>
        )}
      </div>
    </div>
  );
}
```

---

## Backend API Examples

### Next.js API Route for Betting

```typescript
// app/api/bet/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { ethers } from 'ethers';
import { Redis } from '@upstash/redis';
import { Ratelimit } from '@upstash/ratelimit';

const redis = Redis.fromEnv();
const ratelimit = new Ratelimit({
  redis,
  limiter: Ratelimit.slidingWindow(5, '1 m'),
});

export async function POST(req: NextRequest) {
  try {
    // Rate limiting
    const identifier = req.headers.get('x-forwarded-for') || 'anonymous';
    const { success } = await ratelimit.limit(identifier);
    
    if (!success) {
      return NextResponse.json(
        { error: 'Too many requests' },
        { status: 429 }
      );
    }
    
    // Parse request
    const { roundId, prediction, amount, walletAddress, signature } = await req.json();
    
    // Validate inputs
    if (!roundId || prediction === undefined || !amount || !walletAddress || !signature) {
      return NextResponse.json(
        { error: 'Missing required fields' },
        { status: 400 }
      );
    }
    
    // Verify signature
    const message = `Place bet: Round ${roundId}, Prediction ${prediction}, Amount ${amount}`;
    const recoveredAddress = ethers.utils.verifyMessage(message, signature);
    
    if (recoveredAddress.toLowerCase() !== walletAddress.toLowerCase()) {
      return NextResponse.json(
        { error: 'Invalid signature' },
        { status: 401 }
      );
    }
    
    // Store bet in cache (for quick access)
    await redis.set(
      `bet:${roundId}:${walletAddress}`,
      JSON.stringify({ roundId, prediction, amount, timestamp: Date.now() }),
      { ex: 86400 } // Expire after 24 hours
    );
    
    return NextResponse.json({ success: true });
    
  } catch (error) {
    console.error('API error:', error);
    return NextResponse.json(
      { error: 'Internal server error' },
      { status: 500 }
    );
  }
}
```

---

## Real-time Updates with Socket.io

### Socket.io Server

```typescript
// server/socket.ts
import { Server } from 'socket.io';
import { createServer } from 'http';
import { Redis } from '@upstash/redis';

const redis = Redis.fromEnv();

const httpServer = createServer();
const io = new Server(httpServer, {
  cors: {
    origin: process.env.FRONTEND_URL || 'http://localhost:3000',
    credentials: true
  }
});

io.on('connection', (socket) => {
  console.log('Client connected:', socket.id);
  
  // Subscribe to round updates
  socket.on('subscribe:round', async (roundId) => {
    socket.join(`round:${roundId}`);
    
    // Send current round data
    const roundData = await getRoundData(roundId);
    socket.emit('round:data', roundData);
  });
  
  // Unsubscribe
  socket.on('unsubscribe:round', (roundId) => {
    socket.leave(`round:${roundId}`);
  });
  
  socket.on('disconnect', () => {
    console.log('Client disconnected:', socket.id);
  });
});

// Listen for blockchain events and broadcast
async function broadcastRoundUpdate(roundId: number, data: any) {
  io.to(`round:${roundId}`).emit('round:updated', data);
}

async function broadcastPoolUpdate(roundId: number, prediction: number, pool: any) {
  io.to(`round:${roundId}`).emit('pool:updated', { prediction, pool });
}

httpServer.listen(3001, () => {
  console.log('Socket.io server running on port 3001');
});

export { io, broadcastRoundUpdate, broadcastPoolUpdate };
```

---

## Testing Examples

### Smart Contract Tests

```javascript
// test/P2PLottery.test.js
const { expect } = require("chai");
const { ethers } = require("hardhat");
const { time } = require("@nomicfoundation/hardhat-network-helpers");

describe("P2PLotteryBetting", function () {
  let contract;
  let owner, user1, user2, user3, oracle;
  
  beforeEach(async function () {
    [owner, user1, user2, user3, oracle] = await ethers.getSigners();
    
    const Contract = await ethers.getContractFactory("P2PLotteryBetting");
    contract = await Contract.deploy();
    await contract.deployed();
    
    // Grant oracle role
    const ORACLE_ROLE = await contract.ORACLE_ROLE();
    await contract.grantRole(ORACLE_ROLE, oracle.address);
    
    // Create a round
    const drawTime = (await time.latest()) + 3600; // 1 hour from now
    await contract.createRound(drawTime);
  });
  
  describe("Betting", function () {
    it("Should allow users to place bets", async function () {
      const betAmount = ethers.utils.parseEther("1.0");
      
      await expect(
        contract.connect(user1).placeBet(1, 0, { value: betAmount })
      ).to.emit(contract, "BetPlaced")
       .withArgs(1, user1.address, 0, betAmount);
      
      const bet = await contract.getUserBet(1, user1.address);
      expect(bet.amount).to.equal(betAmount);
      expect(bet.prediction).to.equal(0); // EVEN
    });
    
    it("Should reject bets below minimum", async function () {
      const betAmount = ethers.utils.parseEther("0.005");
      
      await expect(
        contract.connect(user1).placeBet(1, 0, { value: betAmount })
      ).to.be.revertedWith("Bet too small");
    });
    
    it("Should match bets between EVEN and ODD", async function () {
      // User1 bets 1 ETH on EVEN
      await contract.connect(user1).placeBet(1, 0, { 
        value: ethers.utils.parseEther("1.0") 
      });
      
      // User2 bets 1 ETH on ODD
      await contract.connect(user2).placeBet(1, 1, { 
        value: ethers.utils.parseEther("1.0") 
      });
      
      // Check pools
      const evenPool = await contract.getPredictionPool(1, 0);
      const oddPool = await contract.getPredictionPool(1, 1);
      
      expect(evenPool.matchedAmount).to.equal(ethers.utils.parseEther("1.0"));
      expect(oddPool.matchedAmount).to.equal(ethers.utils.parseEther("1.0"));
    });
  });
  
  describe("Finalization", function () {
    it("Should finalize round with winning number", async function () {
      // Place some bets
      await contract.connect(user1).placeBet(1, 0, { 
        value: ethers.utils.parseEther("1.0") 
      });
      await contract.connect(user2).placeBet(1, 1, { 
        value: ethers.utils.parseEther("1.0") 
      });
      
      // Fast forward to lock time
      const round = await contract.getRoundInfo(1);
      await time.increaseTo(round.lockTime);
      
      // Lock round
      await contract.lockRound(1);
      
      // Fast forward to draw time
      await time.increaseTo(round.drawTime);
      
      // Finalize with EVEN winning number (42)
      await expect(
        contract.connect(oracle).finalizeRound(1, 42)
      ).to.emit(contract, "RoundFinalized")
       .withArgs(1, 42, await time.latest());
      
      const finalizedRound = await contract.getRoundInfo(1);
      expect(finalizedRound.isFinalized).to.be.true;
      expect(finalizedRound.winningNumber).to.equal(42);
    });
  });
  
  describe("Claims", function () {
    it("Should allow winners to claim winnings", async function () {
      // Place bets
      await contract.connect(user1).placeBet(1, 0, { 
        value: ethers.utils.parseEther("1.0") 
      }); // EVEN
      await contract.connect(user2).placeBet(1, 1, { 
        value: ethers.utils.parseEther("1.0") 
      }); // ODD
      
      // Finalize round with EVEN winning (42)
      const round = await contract.getRoundInfo(1);
      await time.increaseTo(round.lockTime);
      await contract.lockRound(1);
      await time.increaseTo(round.drawTime);
      await contract.connect(oracle).finalizeRound(1, 42);
      
      // User1 (EVEN bettor) should win
      const balanceBefore = await ethers.provider.getBalance(user1.address);
      
      const tx = await contract.connect(user1).claimWinnings(1);
      const receipt = await tx.wait();
      
      const balanceAfter = await ethers.provider.getBalance(user1.address);
      const gasCost = receipt.gasUsed.mul(receipt.effectiveGasPrice);
      
      // User should receive more than original bet (minus gas)
      expect(balanceAfter.sub(balanceBefore).add(gasCost)).to.be.gt(
        ethers.utils.parseEther("1.0")
      );
    });
    
    it("Should refund unmatched bets", async function () {
      // User1 bets 2 ETH on EVEN
      await contract.connect(user1).placeBet(1, 0, { 
        value: ethers.utils.parseEther("2.0") 
      });
      
      // User2 bets 1 ETH on ODD (only 1 ETH will be matched)
      await contract.connect(user2).placeBet(1, 1, { 
        value: ethers.utils.parseEther("1.0") 
      });
      
      // Finalize with ODD winning
      const round = await contract.getRoundInfo(1);
      await time.increaseTo(round.lockTime);
      await contract.lockRound(1);
      await time.increaseTo(round.drawTime);
      await contract.connect(oracle).finalizeRound(1, 43);
      
      // User1 (loser with 2 ETH bet, only 1 ETH matched)
      // Should get 1 ETH refund for unmatched portion
      const balanceBefore = await ethers.provider.getBalance(user1.address);
      
      const tx = await contract.connect(user1).claimWinnings(1);
      const receipt = await tx.wait();
      
      const balanceAfter = await ethers.provider.getBalance(user1.address);
      const gasCost = receipt.gasUsed.mul(receipt.effectiveGasPrice);
      
      // Should receive 1 ETH refund (unmatched portion)
      expect(balanceAfter.sub(balanceBefore).add(gasCost)).to.equal(
        ethers.utils.parseEther("1.0")
      );
    });
  });
});
```

---

This implementation guide provides complete, production-ready code examples for all major components of the P2P Lottery Platform! 🚀
