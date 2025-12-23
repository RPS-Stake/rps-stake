# RPS-Stake Design Specification

## System Architecture Overview

RPS-Stake is built as a multi-chain Web3 application with a hybrid architecture combining on-chain game logic with off-chain AI computation and user interface management.

### Architecture Principles

- **Multi-chain by design:** Deploy identical contracts across 5 major chains
- **Crypto-native:** Pure cryptocurrency payments, no fiat integration
- **Security-first:** Multiple layers of protection for user funds
- **Scalable:** Designed to handle 1000+ concurrent users
- **Responsive:** Mobile-first design with <2s response times

## Technology Stack

### Frontend Stack

- **Framework:** Next.js 14 (App Router)
- **Styling:** Tailwind CSS + shadcn/ui components
- **Web3 Integration:** AppKit (Wagmi/Ethers adapters)
- **State Management:** Zustand for client state
- **UI Components:** Radix UI primitives with custom styling
- **Animations:** Framer Motion for game interactions

### Backend Stack

- **API Layer:** Next.js API Routes (serverless)
- **Database:** PostgreSQL hosted on Supabase
- **Caching:** Redis for session and game state caching
- **Authentication:** Self Protocol integration
- **Monitoring:** Sentry for error tracking, Vercel Analytics

### Blockchain Stack

- **Smart Contracts:** Solidity ^0.8.19
- **Development Framework:** Foundry
- **Security:** OpenZeppelin contracts as base
- **Oracles:** Chainlink for price feeds and VRF
- **Networks:** Ethereum, Base, Celo, Optimism, Arbitrum

## Database Design

### Schema Overview

The database is designed to track on-chain events while providing fast queries for user interfaces. All financial data is primarily stored on-chain with database serving as an indexed cache.

```sql
-- Core user management
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  wallet_address VARCHAR(42) UNIQUE NOT NULL,
  self_verification_status BOOLEAN DEFAULT FALSE,
  verification_timestamp TIMESTAMP,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW(),

  -- Indexes for performance
  INDEX idx_wallet_address (wallet_address),
  INDEX idx_verification_status (self_verification_status)
);

-- Game history tracking (mirrors on-chain events)
CREATE TABLE games (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(id),
  transaction_hash VARCHAR(66) UNIQUE NOT NULL,
  chain_id INTEGER NOT NULL,
  player_move VARCHAR(10) NOT NULL,
  ai_move VARCHAR(10) NOT NULL,
  result VARCHAR(10) NOT NULL, -- 'win', 'lose', 'tie'
  stake_amount DECIMAL(18,8) NOT NULL,
  payout_amount DECIMAL(18,8) DEFAULT 0,
  block_number BIGINT,
  created_at TIMESTAMP DEFAULT NOW(),

  -- Indexes for queries
  INDEX idx_user_games (user_id, created_at DESC),
  INDEX idx_chain_block (chain_id, block_number),
  INDEX idx_tx_hash (transaction_hash)
);

-- Daily limits enforcement
CREATE TABLE daily_limits (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(id),
  chain_id INTEGER NOT NULL,
  date DATE NOT NULL,
  games_played INTEGER DEFAULT 0,
  coins_wagered DECIMAL(18,8) DEFAULT 0,

  UNIQUE(user_id, chain_id, date),
  INDEX idx_user_date_limits (user_id, date)
);

-- Purchase tracking (mirrors on-chain events)
CREATE TABLE token_purchases (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(id),
  transaction_hash VARCHAR(66) UNIQUE NOT NULL,
  chain_id INTEGER NOT NULL,
  token_address VARCHAR(42) NOT NULL,
  token_amount DECIMAL(18,8) NOT NULL,
  coins_received DECIMAL(18,8) NOT NULL,
  usd_value DECIMAL(10,2),
  block_number BIGINT,
  created_at TIMESTAMP DEFAULT NOW(),

  INDEX idx_user_purchases (user_id, created_at DESC),
  INDEX idx_chain_token (chain_id, token_address)
);

-- Cashout tracking
CREATE TABLE cashouts (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(id),
  transaction_hash VARCHAR(66) UNIQUE NOT NULL,
  chain_id INTEGER NOT NULL,
  coins_amount DECIMAL(18,8) NOT NULL,
  token_address VARCHAR(42) NOT NULL,
  token_amount DECIMAL(18,8) NOT NULL,
  status VARCHAR(20) DEFAULT 'pending',
  created_at TIMESTAMP DEFAULT NOW(),

  INDEX idx_user_cashouts (user_id, created_at DESC),
  INDEX idx_status (status)
);
```

### Data Flow Architecture

1. **User Actions** → Smart Contract Transactions
2. **Blockchain Events** → Event Listeners → Database Updates
3. **UI Queries** → Database → Cached Results → Frontend
4. **Real-time Updates** → WebSocket connections for live balance updates

## Smart Contract Architecture

### Core Contract: RPSStake.sol

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import "@openzeppelin/contracts/security/ReentrancyGuard.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/security/Pausable.sol";
import "@chainlink/contracts/src/v0.8/interfaces/AggregatorV3Interface.sol";
import "@chainlink/contracts/src/v0.8/interfaces/VRFCoordinatorV2Interface.sol";

contract RPSStake is ReentrancyGuard, Ownable, Pausable {
    // Events for off-chain tracking
    event CoinsPurchased(
        address indexed user,
        address indexed token,
        uint256 tokenAmount,
        uint256 coins,
        uint256 timestamp
    );

    event GamePlayed(
        address indexed user,
        uint8 playerMove,
        uint8 aiMove,
        uint8 result,
        uint256 stake,
        uint256 payout,
        uint256 timestamp
    );

    event CoinsCashedOut(
        address indexed user,
        address indexed token,
        uint256 coins,
        uint256 tokenAmount,
        uint256 timestamp
    );

    // Game result struct
    struct GameResult {
        uint8 playerMove;    // 0=rock, 1=paper, 2=scissors
        uint8 aiMove;        // 0=rock, 1=paper, 2=scissors
        uint8 result;        // 0=lose, 1=tie, 2=win
        uint256 stake;       // Amount wagered
        uint256 timestamp;   // Block timestamp
    }

    // Supported token configuration
    struct SupportedToken {
        address tokenAddress;
        AggregatorV3Interface priceFeed;
        uint8 decimals;
        bool isActive;
        uint256 minPurchase;  // Minimum purchase amount
        uint256 maxPurchase;  // Maximum purchase amount
    }

    // State variables
    mapping(address => uint256) public rpsCoins;
    mapping(address => GameResult[]) public gameHistory;
    mapping(address => mapping(uint256 => uint256)) public dailyGamesPlayed;
    mapping(address => mapping(uint256 => uint256)) public dailyCoinsWagered;
    mapping(address => SupportedToken) public supportedTokens;

    // Constants
    uint256 public constant COINS_PER_DOLLAR = 10;
    uint256 public constant MAX_DAILY_GAMES = 10;
    uint256 public constant MAX_DAILY_COINS = 50;
    uint256 public constant WIN_MULTIPLIER = 125; // 25% profit (125% of stake)
    uint256 public constant HOUSE_EDGE = 75; // 75% goes to house on losses

    // Special address for ETH
    address public constant ETH_ADDRESS = 0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE;

    // Treasury and fee management
    address public treasury;
    uint256 public totalHouseEarnings;

    // Chainlink VRF for provably fair randomness
    VRFCoordinatorV2Interface public vrfCoordinator;
    uint64 public subscriptionId;
    bytes32 public keyHash;

    // Modifiers
    modifier onlyVerifiedUser() {
        // Integration with Self Protocol verification
        require(isUserVerified(msg.sender), "User not age verified");
        _;
    }

    modifier withinDailyLimits(uint256 stake) {
        uint256 today = block.timestamp / 86400;
        require(dailyGamesPlayed[msg.sender][today] < MAX_DAILY_GAMES, "Daily game limit exceeded");
        require(dailyCoinsWagered[msg.sender][today] + stake <= MAX_DAILY_COINS, "Daily coin limit exceeded");
        _;
    }

    // Core functions
    function buyCoinsWithETH(uint256 coinAmount)
        external
        payable
        nonReentrant
        whenNotPaused
        onlyVerifiedUser
    {
        require(coinAmount > 0, "Invalid coin amount");

        uint256 ethPrice = getTokenPrice(ETH_ADDRESS);
        uint256 requiredETH = calculateTokenAmount(ETH_ADDRESS, coinAmount);
        require(msg.value >= requiredETH, "Insufficient ETH sent");

        // Update user balance
        rpsCoins[msg.sender] += coinAmount;

        // Send excess ETH back
        if (msg.value > requiredETH) {
            payable(msg.sender).transfer(msg.value - requiredETH);
        }

        emit CoinsPurchased(msg.sender, ETH_ADDRESS, requiredETH, coinAmount, block.timestamp);
    }

    function buyCoinsWithToken(address token, uint256 coinAmount)
        external
        nonReentrant
        whenNotPaused
        onlyVerifiedUser
    {
        require(supportedTokens[token].isActive, "Token not supported");
        require(coinAmount > 0, "Invalid coin amount");

        uint256 tokenAmount = calculateTokenAmount(token, coinAmount);
        require(tokenAmount >= supportedTokens[token].minPurchase, "Below minimum purchase");
        require(tokenAmount <= supportedTokens[token].maxPurchase, "Above maximum purchase");

        // Transfer tokens from user
        IERC20(token).transferFrom(msg.sender, address(this), tokenAmount);

        // Update user balance
        rpsCoins[msg.sender] += coinAmount;

        emit CoinsPurchased(msg.sender, token, tokenAmount, coinAmount, block.timestamp);
    }

    function playAgainstAI(uint8 playerMove, uint256 stake)
        external
        nonReentrant
        whenNotPaused
        onlyVerifiedUser
        withinDailyLimits(stake)
    {
        require(playerMove <= 2, "Invalid move");
        require(stake > 0 && stake <= rpsCoins[msg.sender], "Invalid stake amount");

        // Deduct stake
        rpsCoins[msg.sender] -= stake;

        // Get AI move (simplified - in production use Chainlink VRF)
        uint8 aiMove = getAIMove(msg.sender, playerMove);

        // Determine result
        uint8 result = determineWinner(playerMove, aiMove);
        uint256 payout = 0;

        if (result == 2) { // Player wins
            payout = (stake * WIN_MULTIPLIER) / 100;
            rpsCoins[msg.sender] += payout;
        } else if (result == 1) { // Tie
            payout = stake;
            rpsCoins[msg.sender] += payout;
        } else { // Player loses (result == 0)
            // House keeps the stake
            totalHouseEarnings += stake;
        }

        // Update daily limits
        uint256 today = block.timestamp / 86400;
        dailyGamesPlayed[msg.sender][today]++;
        dailyCoinsWagered[msg.sender][today] += stake;

        // Store game result
        gameHistory[msg.sender].push(GameResult({
            playerMove: playerMove,
            aiMove: aiMove,
            result: result,
            stake: stake,
            timestamp: block.timestamp
        }));

        emit GamePlayed(msg.sender, playerMove, aiMove, result, stake, payout, block.timestamp);
    }

    // Cashout functions
    function cashOutETH(uint256 coinAmount)
        external
        nonReentrant
        whenNotPaused
    {
        require(coinAmount > 0 && coinAmount <= rpsCoins[msg.sender], "Invalid coin amount");

        uint256 ethAmount = calculateTokenAmount(ETH_ADDRESS, coinAmount);
        require(address(this).balance >= ethAmount, "Insufficient contract balance");

        // Update user balance
        rpsCoins[msg.sender] -= coinAmount;

        // Transfer ETH
        payable(msg.sender).transfer(ethAmount);

        emit CoinsCashedOut(msg.sender, ETH_ADDRESS, coinAmount, ethAmount, block.timestamp);
    }

    function cashOutToken(address token, uint256 coinAmount)
        external
        nonReentrant
        whenNotPaused
    {
        require(supportedTokens[token].isActive, "Token not supported");
        require(coinAmount > 0 && coinAmount <= rpsCoins[msg.sender], "Invalid coin amount");

        uint256 tokenAmount = calculateTokenAmount(token, coinAmount);
        require(IERC20(token).balanceOf(address(this)) >= tokenAmount, "Insufficient contract balance");

        // Update user balance
        rpsCoins[msg.sender] -= coinAmount;

        // Transfer tokens
        IERC20(token).transfer(msg.sender, tokenAmount);

        emit CoinsCashedOut(msg.sender, token, coinAmount, tokenAmount, block.timestamp);
    }

    // View functions
    function getTokenPrice(address token) public view returns (uint256) {
        if (token == ETH_ADDRESS) {
            // Get ETH/USD price from Chainlink
            (, int256 price, , , ) = AggregatorV3Interface(supportedTokens[token].priceFeed).latestRoundData();
            return uint256(price);
        } else {
            // Get token/USD price from Chainlink
            (, int256 price, , , ) = supportedTokens[token].priceFeed.latestRoundData();
            return uint256(price);
        }
    }

    function calculateTokenAmount(address token, uint256 coinAmount) public view returns (uint256) {
        uint256 usdAmount = coinAmount * 1e8 / COINS_PER_DOLLAR; // USD amount in 8 decimals
        uint256 tokenPrice = getTokenPrice(token); // Price in 8 decimals
        uint256 tokenDecimals = supportedTokens[token].decimals;

        return (usdAmount * 10**tokenDecimals) / tokenPrice;
    }

    function getDailyStats(address user) external view returns (uint256 games, uint256 wagered) {
        uint256 today = block.timestamp / 86400;
        return (dailyGamesPlayed[user][today], dailyCoinsWagered[user][today]);
    }

    function getUserGameHistory(address user, uint256 limit)
        external
        view
        returns (GameResult[] memory)
    {
        GameResult[] storage userGames = gameHistory[user];
        uint256 length = userGames.length;
        uint256 returnLength = limit > length ? length : limit;

        GameResult[] memory recentGames = new GameResult[](returnLength);

        for (uint256 i = 0; i < returnLength; i++) {
            recentGames[i] = userGames[length - 1 - i]; // Most recent first
        }

        return recentGames;
    }

    // Internal functions
    function determineWinner(uint8 playerMove, uint8 aiMove) internal pure returns (uint8) {
        if (playerMove == aiMove) return 1; // Tie

        // Rock(0) beats Scissors(2), Paper(1) beats Rock(0), Scissors(2) beats Paper(1)
        if ((playerMove == 0 && aiMove == 2) ||
            (playerMove == 1 && aiMove == 0) ||
            (playerMove == 2 && aiMove == 1)) {
            return 2; // Player wins
        }

        return 0; // AI wins
    }

    function getAIMove(address player, uint8 playerMove) internal view returns (uint8) {
        // Simplified AI - in production, use more sophisticated algorithm
        // This should integrate with off-chain AI engine
        uint256 randomness = uint256(keccak256(abi.encodePacked(
            block.timestamp,
            block.difficulty,
            player,
            playerMove
        )));

        return uint8(randomness % 3);
    }

    function isUserVerified(address user) internal view returns (bool) {
        // Integration point with Self Protocol
        // This should call Self Protocol verification contract
        return true; // Placeholder
    }

    // Admin functions
    function addSupportedToken(
        address token,
        address priceFeed,
        uint8 decimals,
        uint256 minPurchase,
        uint256 maxPurchase
    ) external onlyOwner {
        supportedTokens[token] = SupportedToken({
            tokenAddress: token,
            priceFeed: AggregatorV3Interface(priceFeed),
            decimals: decimals,
            isActive: true,
            minPurchase: minPurchase,
            maxPurchase: maxPurchase
        });
    }

    function removeSupportedToken(address token) external onlyOwner {
        supportedTokens[token].isActive = false;
    }

    function setTreasury(address _treasury) external onlyOwner {
        treasury = _treasury;
    }

    function withdrawHouseEarnings() external onlyOwner {
        require(treasury != address(0), "Treasury not set");
        // Convert house earnings to treasury
        // Implementation depends on tokenomics
    }

    function emergencyPause() external onlyOwner {
        _pause();
    }

    function unpause() external onlyOwner {
        _unpause();
    }

    // Receive function for ETH deposits
    receive() external payable {}
}
```

### Multi-Chain Deployment Strategy

```solidity
// Deployment configuration for each chain
struct ChainConfig {
    uint256 chainId;
    string name;
    address[] supportedTokens;
    address[] priceFeeds;
    address vrfCoordinator;
    bytes32 keyHash;
}

// Chain configurations
ChainConfig[] public chains = [
    ChainConfig({
        chainId: 1,
        name: "Ethereum",
        supportedTokens: [ETH_ADDRESS, USDC_ADDRESS, USDT_ADDRESS],
        priceFeeds: [ETH_USD_FEED, USDC_USD_FEED, USDT_USD_FEED],
        vrfCoordinator: 0x...,
        keyHash: 0x...
    }),
    ChainConfig({
        chainId: 8453,
        name: "Base",
        supportedTokens: [ETH_ADDRESS, USDC_ADDRESS],
        priceFeeds: [ETH_USD_FEED, USDC_USD_FEED],
        vrfCoordinator: 0x...,
        keyHash: 0x...
    }),
    // ... other chains
];
```

## AI Engine Architecture

### AI Strategy Framework

```typescript
interface AIStrategy {
  analyzePattern(playerHistory: Move[]): PatternAnalysis;
  selectMove(analysis: PatternAnalysis, difficulty: AIDifficulty): Move;
  updateModel(gameResult: GameResult): void;
}

class RPSAIEngine implements AIStrategy {
  private patternRecognizer: PatternRecognizer;
  private winRateManager: WinRateManager;
  private psychologyEngine: PsychologyEngine;

  constructor() {
    this.patternRecognizer = new PatternRecognizer();
    this.winRateManager = new WinRateManager(0.75); // 75% target win rate
    this.psychologyEngine = new PsychologyEngine();
  }

  analyzePattern(playerHistory: Move[]): PatternAnalysis {
    return {
      sequences: this.patternRecognizer.findSequences(playerHistory),
      frequency: this.patternRecognizer.getMoveFrequency(playerHistory),
      streaks: this.patternRecognizer.getStreaks(playerHistory),
      predictedNext: this.patternRecognizer.predictNext(playerHistory),
    };
  }

  selectMove(analysis: PatternAnalysis, difficulty: AIDifficulty): Move {
    // Combine multiple strategies based on difficulty
    const strategies = [
      this.patternRecognizer.counterStrategy(analysis),
      this.psychologyEngine.exploitBias(analysis),
      this.winRateManager.adjustForTarget(analysis),
      this.randomStrategy(),
    ];

    return this.combineStrategies(strategies, difficulty);
  }
}
```

### Pattern Recognition System

```typescript
class PatternRecognizer {
  private readonly SEQUENCE_LENGTH = 5;
  private patterns: Map<string, Move> = new Map();

  findSequences(history: Move[]): SequencePattern[] {
    const sequences: SequencePattern[] = [];

    for (let i = 0; i <= history.length - this.SEQUENCE_LENGTH; i++) {
      const sequence = history.slice(i, i + this.SEQUENCE_LENGTH);
      const key = sequence.join("");

      if (this.patterns.has(key)) {
        sequences.push({
          pattern: sequence,
          frequency: this.getPatternFrequency(key, history),
          predictedNext: this.patterns.get(key)!,
        });
      }
    }

    return sequences.sort((a, b) => b.frequency - a.frequency);
  }

  predictNext(history: Move[]): Move {
    if (history.length < 3) return this.randomMove();

    // Look for recent patterns
    const recentMoves = history.slice(-4);
    const pattern = recentMoves.join("");

    if (this.patterns.has(pattern)) {
      return this.counterMove(this.patterns.get(pattern)!);
    }

    // Fallback to frequency analysis
    return this.counterMostFrequent(history);
  }

  private counterMove(move: Move): Move {
    const counters = {
      rock: "paper",
      paper: "scissors",
      scissors: "rock",
    };
    return counters[move] as Move;
  }
}
```

## API Architecture

### RESTful API Design

```typescript
// API Route Structure
/api/
├── auth/
│   ├── verify              # POST - Self Protocol verification
│   └── status              # GET - Check auth status
├── user/
│   ├── profile             # GET - User profile and balances
│   ├── history             # GET - Game history with pagination
│   ├── stats               # GET - User statistics
│   └── daily-limits        # GET - Current daily limits
├── blockchain/
│   ├── sync-events         # POST - Sync blockchain events
│   ├── token-prices        # GET - Current token prices
│   ├── estimate-gas        # POST - Gas estimation
│   └── supported-tokens    # GET - Supported tokens per chain
├── game/
│   ├── play                # POST - Initiate AI game
│   ├── ai-move             # GET - Get AI move
│   └── verify-result       # POST - Verify game result
└── admin/
    ├── stats               # GET - Platform statistics
    ├── users               # GET - User management
    └── treasury            # GET - Treasury status
```

### API Response Standards

```typescript
// Standard API Response Format
interface APIResponse<T> {
  success: boolean;
  data?: T;
  error?: {
    code: string;
    message: string;
    details?: any;
  };
  meta?: {
    timestamp: string;
    requestId: string;
    pagination?: PaginationMeta;
  };
}

// Error Codes
enum APIErrorCodes {
  VALIDATION_ERROR = "VALIDATION_ERROR",
  AUTHENTICATION_REQUIRED = "AUTH_REQUIRED",
  INSUFFICIENT_BALANCE = "INSUFFICIENT_BALANCE",
  DAILY_LIMIT_EXCEEDED = "DAILY_LIMIT_EXCEEDED",
  BLOCKCHAIN_ERROR = "BLOCKCHAIN_ERROR",
  INTERNAL_ERROR = "INTERNAL_ERROR",
}
```

## Security Architecture

### Multi-Layer Security Model

1. **Smart Contract Security**

   - OpenZeppelin security patterns
   - Reentrancy guards on all external calls
   - Access control with role-based permissions
   - Emergency pause functionality
   - Multi-signature treasury management

2. **Application Security**

   - Rate limiting on all API endpoints
   - Input validation and sanitization
   - CORS configuration
   - HTTPS enforcement
   - Environment variable protection

3. **Financial Security**
   - Daily withdrawal limits
   - Suspicious activity monitoring
   - Cold storage for majority of funds
   - Real-time transaction monitoring

### Security Monitoring

```typescript
// Security Event Monitoring
interface SecurityEvent {
  type: "SUSPICIOUS_ACTIVITY" | "RATE_LIMIT_EXCEEDED" | "LARGE_TRANSACTION";
  userId: string;
  details: any;
  timestamp: Date;
  severity: "LOW" | "MEDIUM" | "HIGH" | "CRITICAL";
}

class SecurityMonitor {
  async detectSuspiciousActivity(
    userId: string,
    action: string
  ): Promise<void> {
    // Implement detection logic
    const riskScore = await this.calculateRiskScore(userId, action);

    if (riskScore > RISK_THRESHOLD) {
      await this.triggerSecurityAlert({
        type: "SUSPICIOUS_ACTIVITY",
        userId,
        details: { action, riskScore },
        timestamp: new Date(),
        severity: this.getSeverity(riskScore),
      });
    }
  }
}
```

## Performance Optimization

### Caching Strategy

```typescript
// Multi-level caching
interface CacheStrategy {
  // Level 1: In-memory cache (Redis)
  userBalances: TTL_5_MINUTES;
  tokenPrices: TTL_1_MINUTE;
  gameHistory: TTL_10_MINUTES;

  // Level 2: Database query optimization
  indexedQueries: true;
  connectionPooling: true;

  // Level 3: CDN caching
  staticAssets: TTL_1_DAY;
  apiResponses: TTL_30_SECONDS;
}
```

### Database Optimization

```sql
-- Performance indexes
CREATE INDEX CONCURRENTLY idx_games_user_recent
ON games (user_id, created_at DESC)
WHERE created_at > NOW() - INTERVAL '30 days';

CREATE INDEX CONCURRENTLY idx_daily_limits_active
ON daily_limits (user_id, date)
WHERE date >= CURRENT_DATE - INTERVAL '7 days';

-- Partitioning for large tables
CREATE TABLE games_y2024m01 PARTITION OF games
FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');
```

## Monitoring & Observability

### Metrics Collection

```typescript
// Key Performance Indicators
interface PlatformMetrics {
  // Business Metrics
  dailyActiveUsers: number;
  totalRevenue: number;
  averageSessionDuration: number;
  userRetentionRate: number;

  // Technical Metrics
  apiResponseTime: number;
  errorRate: number;
  uptime: number;
  transactionSuccessRate: number;

  // Game Metrics
  gamesPlayed: number;
  averageStakeSize: number;
  houseEdgeActual: number;
  aiWinRate: number;
}
```

### Alerting System

```typescript
// Alert Configuration
const alerts = {
  criticalErrors: {
    threshold: 1,
    window: "1m",
    channels: ["slack", "email", "sms"],
  },
  highErrorRate: {
    threshold: 0.05, // 5%
    window: "5m",
    channels: ["slack", "email"],
  },
  lowBalance: {
    threshold: 1000, // $1000 equivalent
    window: "1h",
    channels: ["slack"],
  },
};
```

This design specification provides a comprehensive technical foundation for RPS-Stake Phase 1, ensuring scalability, security, and maintainability while focusing on the crypto-only implementation approach.
