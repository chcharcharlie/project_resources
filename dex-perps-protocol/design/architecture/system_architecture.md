# System Architecture Document - DEX Perpetual Contracts Protocol

## 1. Executive Summary

This document presents the technical architecture for a decentralized perpetual contracts exchange targeting retail traders. The architecture is specifically designed for AI-driven implementation, featuring larger, self-contained components that can be fully reasoned about and implemented within Claude's context window.

### Key Architecture Decisions
- **Blockchain Platform**: Polygon (primary recommendation) with Arbitrum as alternative
- **Smart Contract Language**: Solidity 0.8.19+
- **Architecture Pattern**: Modular upgradeable contracts with clear separation of concerns
- **Component Design**: AI-optimized with complete functional units
- **Security Model**: Multi-layered with timelock controls and circuit breakers

## 2. Technology Stack Selection

### 2.1 Blockchain Platform Analysis

**Primary Recommendation: Polygon**
- **Pros**:
  - 2-second block times suitable for derivatives trading
  - $0.01-0.05 transaction costs accessible for retail
  - Mature DeFi ecosystem with existing oracle infrastructure
  - EVM compatibility for familiar tooling
  - High throughput (65,000 TPS theoretical)
- **Cons**:
  - Not as decentralized as Ethereum L1
  - Occasional network congestion during high activity

**Alternative: Arbitrum**
- **Pros**:
  - Strong security inherited from Ethereum
  - Growing DeFi ecosystem
  - Lower fees than Ethereum L1
  - Better decentralization than Polygon
- **Cons**:
  - Slightly higher fees than Polygon
  - Newer ecosystem

**Not Recommended: Solana**
- While performant, requires complete architecture redesign
- Different programming model (Rust/Anchor)
- Less mature perpetuals infrastructure

### 2.2 Core Technology Stack

```
Layer                 | Technology              | Justification
---------------------|-------------------------|----------------------------------
Smart Contracts      | Solidity 0.8.19+        | Security features, gas optimizations
Development          | Hardhat                 | Best testing and debugging tools
Testing              | Foundry + Hardhat       | Comprehensive testing coverage
Oracle               | Chainlink + Pyth        | Multi-oracle redundancy
Frontend SDK         | ethers.js v6            | Modern, well-maintained
Indexing             | The Graph               | Decentralized query infrastructure
Monitoring           | OpenZeppelin Defender   | Security monitoring
```

### 2.3 Smart Contract Libraries

```solidity
// Core Dependencies
OpenZeppelin Contracts 4.9+    // Security-audited base contracts
PRBMath                       // Fixed-point math operations
solmate                       // Gas-optimized alternatives
```

## 3. High-Level Architecture

### 3.1 System Overview

```
┌─────────────────────────────────────────────────────────────┐
│                        Users (Traders)                       │
└─────────────────┬───────────────────────┬───────────────────┘
                  │                       │
         ┌────────▼────────┐     ┌────────▼────────┐
         │ Trading Engine  │     │ Risk Management │
         │    Module       │     │    Module       │
         └────────┬────────┘     └────────┬────────┘
                  │                       │
         ┌────────▼───────────────────────▼────────┐
         │          Core Protocol Engine           │
         │  (Position, Margin, Funding, Orders)    │
         └────────┬───────────────────────┬────────┘
                  │                       │
         ┌────────▼────────┐     ┌────────▼────────┐
         │ Oracle Module   │     │ Insurance Fund  │
         └─────────────────┘     └─────────────────┘
```

### 3.2 Component Architecture

The system is divided into six major implementation units, each designed to be complete and self-contained for AI implementation:

1. **Trading Engine Module** (~3000 lines)
   - Order book management
   - Order matching engine
   - Trade execution

2. **Core Protocol Engine** (~4000 lines)
   - Position management
   - Margin calculations
   - Funding rate system
   - P&L tracking

3. **Risk Management Module** (~2500 lines)
   - Liquidation engine
   - Circuit breakers
   - Position limits

4. **Oracle Integration Module** (~1500 lines)
   - Multi-oracle aggregation
   - Price validation
   - Fallback mechanisms

5. **Insurance Fund Module** (~1000 lines)
   - Bad debt coverage
   - Fund management
   - ADL implementation

6. **Market Registry Module** (~1000 lines)
   - Market configuration
   - Parameter management
   - Access control

## 4. Detailed Component Design

### 4.1 Trading Engine Module

**Purpose**: Handle all order-related operations including placement, cancellation, and matching.

**Subcomponents**:
```
TradingEngine/
├── OrderBook.sol         // Order storage and retrieval
├── MatchingEngine.sol    // Order matching logic
├── OrderValidator.sol    // Order validation rules
└── TradingEvents.sol     // Event definitions
```

**Key Interfaces**:
```solidity
interface ITradingEngine {
    // Order Management
    function placeOrder(
        address market,
        OrderType orderType,
        Side side,
        uint256 price,
        uint256 size,
        TimeInForce tif
    ) external returns (uint256 orderId);
    
    function cancelOrder(uint256 orderId) external;
    function cancelAllOrders(address market) external;
    
    // Order Matching
    function matchOrders(address market, uint256 maxMatches) external;
    
    // Views
    function getOrderBook(address market, uint256 depth) 
        external view returns (OrderBookView memory);
    function getOrder(uint256 orderId) 
        external view returns (Order memory);
}
```

**Data Structures**:
```solidity
struct Order {
    uint256 orderId;
    address trader;
    address market;
    OrderType orderType;
    Side side;
    uint256 price;
    uint256 size;
    uint256 filled;
    uint256 timestamp;
    OrderStatus status;
    TimeInForce timeInForce;
}

struct OrderBook {
    // Bid side - sorted high to low
    mapping(uint256 => DoublyLinkedList.List) bids;
    uint256[] bidPrices;
    
    // Ask side - sorted low to high  
    mapping(uint256 => DoublyLinkedList.List) asks;
    uint256[] askPrices;
    
    // Order lookup
    mapping(uint256 => Order) orders;
    mapping(address => uint256[]) userOrders;
}
```

**Implementation Strategy**:
- Use sorted linked lists for price levels
- Binary search for price insertion
- Lazy deletion for gas efficiency
- Batch matching in single transaction

### 4.2 Core Protocol Engine

**Purpose**: Manage positions, margins, funding rates, and P&L calculations.

**Subcomponents**:
```
CoreProtocol/
├── PositionManager.sol      // Position lifecycle
├── MarginManager.sol        // Margin calculations
├── FundingRateEngine.sol    // Funding rate logic
├── PnLCalculator.sol        // P&L computations
└── CollateralManager.sol    // Multi-collateral support
```

**Key Interfaces**:
```solidity
interface ICoreProtocol {
    // Position Management
    function openPosition(
        address trader,
        address market,
        int256 sizeDelta,
        uint256 price
    ) external returns (bytes32 positionKey);
    
    function closePosition(
        address trader,
        address market,
        int256 sizeDelta
    ) external returns (int256 realizedPnL);
    
    // Margin Operations
    function addMargin(
        address market,
        uint256 amount,
        address collateral
    ) external;
    
    function removeMargin(
        address market,
        uint256 amount
    ) external;
    
    // Funding
    function settleFunding(address market) external;
    
    // Views
    function getPosition(address trader, address market)
        external view returns (Position memory);
    function getAccountValue(address trader)
        external view returns (uint256 value, uint256 margin);
}
```

**Position Management**:
```solidity
struct Position {
    int256 size;              // Positive = long, negative = short
    uint256 margin;           // Deposited margin
    uint256 avgEntryPrice;    // Volume-weighted entry
    uint256 lastFundingIndex; // For funding settlement
    uint256 openTimestamp;
    
    // Cached calculations
    int256 unrealizedPnL;
    uint256 liquidationPrice;
    uint256 marginRatio;
}

// Efficient storage using single mapping
mapping(bytes32 => Position) positions;
// positionKey = keccak256(trader, market)
```

**Funding Rate Calculation**:
```solidity
struct FundingRate {
    int256 rate;              // Current funding rate
    uint256 timestamp;        // Last update time
    uint256 cumulativeIndex;  // For position settlement
    
    // Premium calculation inputs
    uint256 markPrice;
    uint256 indexPrice;
    uint256 premiumEMA;      // Exponential moving average
}

// Modified funding formula with 3x adjustment
fundingRate = (premium * 3 + interestRate) * timeElapsed / fundingInterval;
fundingRate = clamp(fundingRate, -maxFundingRate, maxFundingRate);
```

### 4.3 Risk Management Module  

**Purpose**: Monitor and manage system-wide risk through liquidations, limits, and circuit breakers.

**Subcomponents**:
```
RiskManagement/
├── LiquidationEngine.sol    // Liquidation logic
├── CircuitBreaker.sol       // Emergency controls
├── PositionLimits.sol       // Risk limits
└── RiskCalculator.sol       // Risk metrics
```

**Liquidation Engine Design**:
```solidity
interface ILiquidationEngine {
    function liquidatePosition(
        address trader,
        address market
    ) external returns (uint256 liquidatedSize, uint256 penalty);
    
    function isLiquidatable(address trader, address market)
        external view returns (bool);
        
    function getLiquidationPrice(address trader, address market)
        external view returns (uint256);
}

struct LiquidationParams {
    uint256 partialLiquidationThreshold;  // 50,000 USD
    uint256 partialLiquidationRatio;      // 50%
    uint256 liquidationPenalty;           // 1.5%
    uint256 liquidatorReward;             // 0.5%
    uint256 insuranceFundShare;           // 1.0%
}
```

**Circuit Breaker Implementation**:
```solidity
struct CircuitBreaker {
    uint256 priceDeviationThreshold5m;    // 5%
    uint256 priceDeviationThreshold1h;    // 10%
    uint256 priceDeviationThreshold24h;   // 20%
    uint256 cooldownPeriod;               // 5 minutes
    
    mapping(address => TripStatus) marketStatus;
}

enum TripStatus {
    Normal,
    Warning,
    Halted,
    Cooldown
}
```

### 4.4 Oracle Integration Module

**Purpose**: Aggregate prices from multiple sources with validation and fallback mechanisms.

**Subcomponents**:
```
OracleIntegration/
├── OracleAggregator.sol     // Price aggregation
├── OracleValidator.sol      // Validation logic
├── ChainlinkAdapter.sol     // Chainlink integration
├── PythAdapter.sol          // Pyth integration
└── FallbackOracle.sol       // Emergency fallback
```

**Aggregation Strategy**:
```solidity
interface IOracleAggregator {
    function getPrice(address market) 
        external view returns (
            uint256 price,
            uint256 confidence,
            uint256 timestamp
        );
        
    function addOracle(
        address market,
        address oracle,
        uint256 weight
    ) external;
}

struct PriceData {
    uint256 price;
    uint256 confidence;
    uint256 timestamp;
    address source;
}

// Aggregation algorithm
function aggregatePrice(PriceData[] memory prices) internal pure returns (uint256) {
    // 1. Calculate median
    uint256 median = calculateMedian(prices);
    
    // 2. Filter outliers (MAD method)
    PriceData[] memory filtered = filterOutliers(prices, median);
    
    // 3. Weighted average of remaining
    return calculateWeightedAverage(filtered);
}
```

### 4.5 Insurance Fund Module

**Purpose**: Manage the insurance fund for bad debt coverage and system stability.

**Subcomponents**:
```
InsuranceFund/
├── InsuranceFund.sol        // Fund management
├── AutoDeleveraging.sol     // ADL mechanism
└── FundAllocator.sol        // Revenue distribution
```

**Insurance Fund Design**:
```solidity
interface IInsuranceFund {
    function coverBadDebt(
        address market,
        uint256 amount
    ) external returns (bool covered);
    
    function addRevenue(
        address source,
        uint256 amount
    ) external;
    
    function getBalance(address market) 
        external view returns (uint256);
}

struct FundAllocation {
    uint256 tradingFees;      // 40% of fees
    uint256 liquidationFees;  // 1% penalties
    uint256 directFunding;    // Protocol seeding
}
```

### 4.6 Market Registry Module

**Purpose**: Manage market configurations and system parameters.

**Subcomponents**:
```
MarketRegistry/
├── MarketRegistry.sol       // Market management
├── ParameterStore.sol       // Configuration storage
├── AccessController.sol     // Permission management
└── EmergencyAdmin.sol       // Emergency controls
```

**Market Configuration**:
```solidity
struct MarketConfig {
    // Trading parameters
    uint256 tickSize;
    uint256 lotSize;
    uint256 maxLeverage;
    
    // Risk parameters
    uint256 initialMarginRate;
    uint256 maintenanceMarginRate;
    uint256 maxPositionSize;
    uint256 maxOrderSize;
    
    // Fee structure
    int256 makerFee;          // Can be negative (rebate)
    uint256 takerFee;
    
    // Oracle configuration
    address[] oracles;
    uint256[] weights;
    
    // Status
    bool isActive;
    bool isFrozen;
}
```

## 5. Component Dependencies and Interfaces

### 5.1 Dependency Graph

```
MarketRegistry ─────────────────────────┐
     │                                   │
     ├──► TradingEngine ◄────────────────┤
     │         │                         │
     │         ▼                         │
     ├──► CoreProtocol ◄─────────────────┤
     │         │                         │
     │         ▼                         │
     ├──► RiskManagement                 │
     │         │                         │
     │         ▼                         │
     ├──► OracleIntegration              │
     │         │                         │
     │         ▼                         │
     └──► InsuranceFund ◄────────────────┘
```

### 5.2 Interface Definitions

**Cross-Module Communication**:
```solidity
// Shared interfaces for module communication
interface IMarketRegistry {
    function getMarketConfig(address market) 
        external view returns (MarketConfig memory);
    function isMarketActive(address market) 
        external view returns (bool);
}

interface IPriceProvider {
    function getMarkPrice(address market) 
        external view returns (uint256);
    function getIndexPrice(address market) 
        external view returns (uint256);
}

interface IPositionProvider {
    function getPosition(address trader, address market)
        external view returns (Position memory);
    function getAccountMargin(address trader)
        external view returns (uint256);
}
```

## 6. Data Architecture

### 6.1 Storage Optimization

**Packing Strategy**:
```solidity
// Efficient 3-slot position storage
struct PackedPosition {
    int128 size;           // Slot 1: 16 bytes
    uint128 margin;        // Slot 1: 16 bytes (total: 32)
    
    uint128 entryPrice;    // Slot 2: 16 bytes  
    uint64 fundingIndex;   // Slot 2: 8 bytes
    uint64 timestamp;      // Slot 2: 8 bytes (total: 32)
    
    int128 realizedPnL;    // Slot 3: 16 bytes
    uint128 reserved;      // Slot 3: 16 bytes (total: 32)
}
```

**Storage Patterns**:
```solidity
// Single mapping for all positions
mapping(bytes32 => PackedPosition) private positions;

// Efficient order storage with enumeration
mapping(uint256 => Order) private orders;
mapping(address => EnumerableSet.UintSet) private userOrderIds;

// Market data with single access point
mapping(address => MarketData) private markets;
```

### 6.2 Event Architecture

**Event Categories**:
1. **Trading Events**: Order placement, cancellation, execution
2. **Position Events**: Open, close, modify, liquidate
3. **Risk Events**: Circuit breaker, liquidation, ADL
4. **System Events**: Market updates, parameter changes

**Event Design**:
```solidity
// Indexed for efficient filtering
event OrderPlaced(
    address indexed trader,
    address indexed market,
    uint256 indexed orderId,
    OrderData data
);

// Comprehensive data in single event
struct OrderData {
    OrderType orderType;
    Side side;
    uint256 price;
    uint256 size;
    uint256 timestamp;
}
```

## 7. Security Architecture

### 7.1 Security Layers

**Layer 1: Access Control**
```solidity
// Role-based permissions
bytes32 constant ADMIN_ROLE = keccak256("ADMIN");
bytes32 constant OPERATOR_ROLE = keccak256("OPERATOR");
bytes32 constant LIQUIDATOR_ROLE = keccak256("LIQUIDATOR");
bytes32 constant PAUSER_ROLE = keccak256("PAUSER");
```

**Layer 2: Validation**
- Input validation on all external functions
- Overflow protection with SafeMath/checked math
- Reentrancy guards on state-changing functions

**Layer 3: Emergency Controls**
```solidity
// Timelock for critical operations
uint256 constant TIMELOCK_DELAY = 48 hours;

// Circuit breakers
bool public tradingPaused;
mapping(address => bool) public marketPaused;

// Emergency settlement
function emergencySettle(address market) external onlyAdmin {
    require(emergencyMode, "Not in emergency");
    // Force settle all positions at oracle price
}
```

### 7.2 Upgrade Strategy

**Proxy Pattern**:
```solidity
// OpenZeppelin TransparentUpgradeableProxy
// Separate ProxyAdmin contract
// Storage gaps for future additions

contract TradingEngineV1 is Initializable, UUPSUpgradeable {
    // Storage variables...
    
    // Reserve storage slots
    uint256[50] private __gap;
}
```

## 8. Implementation Plan

### 8.1 Component Implementation Order

1. **Phase 1: Foundation** (Week 1-2)
   - Market Registry Module
   - Basic Oracle Integration
   - Core data structures

2. **Phase 2: Trading Core** (Week 3-4)
   - Trading Engine Module
   - Order book implementation
   - Basic matching engine

3. **Phase 3: Position Management** (Week 5-6)
   - Core Protocol Engine
   - Position tracking
   - Margin calculations

4. **Phase 4: Risk Systems** (Week 7-8)
   - Risk Management Module
   - Liquidation engine
   - Circuit breakers

5. **Phase 5: Advanced Features** (Week 9-10)
   - Insurance Fund Module
   - Funding rate system
   - Multi-collateral support

6. **Phase 6: Integration** (Week 11-12)
   - System integration
   - End-to-end testing
   - Performance optimization

### 8.2 AI Implementation Guidelines

**Component Sizing**:
- Each module designed for 15-30 minute implementation sessions
- Complete functionality within each component
- Minimal cross-component dependencies
- Clear interfaces between modules

**Implementation Order Per Component**:
1. Define data structures
2. Implement core logic
3. Add validation and security
4. Create unit tests
5. Optimize for gas

## 9. Performance Optimization

### 9.1 Gas Optimization Strategies

**Storage**:
- Pack structs to minimize slots
- Use mappings over arrays where possible
- Implement lazy deletion
- Cache frequently accessed values

**Computation**:
- Pre-calculate when possible
- Use assembly for critical paths
- Batch operations
- Minimize external calls

**Example Optimizations**:
```solidity
// Batch order matching
function matchOrdersBatch(
    address market,
    uint256 maxMatches
) external returns (uint256 matched) {
    // Process multiple matches in one transaction
    while (matched < maxMatches && hasMatchableOrders(market)) {
        matched += matchSingleOrder(market);
    }
}

// Efficient position updates
function updatePosition(
    bytes32 key,
    int256 sizeDelta,
    uint256 price
) internal {
    Position storage pos = positions[key];
    
    // Single SSTORE by updating all fields
    int256 newSize = pos.size + sizeDelta;
    uint256 newAvgPrice = calculateAvgPrice(pos, sizeDelta, price);
    
    pos.size = newSize;
    pos.avgEntryPrice = newAvgPrice;
    pos.lastUpdate = block.timestamp;
}
```

### 9.2 Scalability Considerations

**On-chain Limitations**:
- Order book depth limited to top N levels
- Batch processing for high volume
- Off-chain order matching consideration for future

**State Management**:
- Regular cleanup of expired orders
- Archival of old positions
- Efficient indexing structures

## 10. Testing Architecture

### 10.1 Test Categories

1. **Unit Tests**: Each function in isolation
2. **Integration Tests**: Module interactions
3. **Scenario Tests**: Complete user flows
4. **Stress Tests**: High volume scenarios
5. **Security Tests**: Attack vectors

### 10.2 Test Implementation

```solidity
// Foundry test example
contract TradingEngineTest is Test {
    TradingEngine engine;
    
    function setUp() public {
        engine = new TradingEngine();
        // Setup test environment
    }
    
    function testPlaceOrder() public {
        // Test order placement
        uint256 orderId = engine.placeOrder(...);
        assertEq(engine.getOrder(orderId).status, OrderStatus.Active);
    }
    
    function testFuzzOrderMatching(
        uint256 price,
        uint256 size
    ) public {
        // Fuzz testing for edge cases
        price = bound(price, MIN_PRICE, MAX_PRICE);
        size = bound(size, MIN_SIZE, MAX_SIZE);
        // Test with random inputs
    }
}
```

## 11. Deployment Architecture

### 11.1 Deployment Strategy

**Testnet Deployment**:
1. Deploy to Polygon Mumbai
2. Initialize with test parameters
3. Run public beta (4 weeks)
4. Collect performance metrics

**Mainnet Deployment**:
1. Deploy proxy contracts
2. Initialize with conservative parameters
3. Gradual parameter relaxation
4. Progressive decentralization

### 11.2 Configuration Management

```solidity
// Deployment configuration
struct DeploymentConfig {
    // Initial markets
    string[] symbols;
    uint256[] leverages;
    
    // Risk parameters
    uint256 maxPositionSize;
    uint256 liquidationPenalty;
    
    // Fee structure
    int256 makerFee;
    uint256 takerFee;
    
    // Oracle configuration
    address chainlinkPriceFeeds;
    address pythPriceFeeds;
}
```

## 12. Monitoring and Maintenance

### 12.1 Monitoring Requirements

**On-chain Monitoring**:
- Position health metrics
- Order book depth
- Funding rate calculations
- Oracle price deviations

**Off-chain Monitoring**:
- Gas usage patterns
- Transaction success rates
- Liquidation efficiency
- System performance metrics

### 12.2 Maintenance Procedures

**Regular Maintenance**:
- Parameter adjustments based on market conditions
- Oracle weight rebalancing
- Risk parameter tuning
- Performance optimization

**Emergency Procedures**:
- Circuit breaker activation
- Emergency position settlement
- Oracle failover
- System pause/unpause

---

This architecture document provides a complete technical blueprint for implementing the DEX Perpetual Contracts Protocol. The design prioritizes AI implementation efficiency while maintaining professional-grade system architecture suitable for high-volume derivatives trading.