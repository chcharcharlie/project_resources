# Product Requirements Document (PRD) - DEX Perpetual Contracts Protocol

## Table of Contents
1. [Product Overview](#product-overview)
2. [Feature Specifications](#feature-specifications)
3. [User Workflows](#user-workflows)
4. [Technical Architecture](#technical-architecture)
5. [Data Models](#data-models)
6. [Non-Functional Requirements](#non-functional-requirements)
7. [Implementation Guidelines](#implementation-guidelines)
8. [Appendices](#appendices)

---

## 1. Product Overview

### 1.1 Product Vision
The DEX Perpetual Contracts Protocol aims to democratize access to leveraged US equity trading through a decentralized, on-chain perpetual futures exchange. Positioned as the "perp DEX version of Robinhood," the protocol enables retail crypto traders to gain exposure to stock market movements with up to 100X leverage, all while maintaining full decentralization and self-custody of funds.

### 1.2 Core Value Proposition
- **Accessibility**: Retail-friendly access to US equity exposure through crypto
- **High Leverage**: Up to 100X leverage for capital efficiency
- **Decentralization**: Fully on-chain execution with no centralized dependencies
- **24/7 Trading**: Trade US stock perpetuals anytime, not limited to market hours
- **Self-Custody**: Users maintain control of their funds at all times

### 1.3 Success Criteria
- Achieve $5B daily trading volume within 12 months
- Maintain perpetual-spot price deviation under 0.1%
- Process 1000+ orders per second
- Achieve <1% bad debt ratio
- 99.9% system uptime

## 2. Feature Specifications

### 2.1 Order Book System

#### 2.1.1 Order Types

**Market Orders**
- **Definition**: Orders that execute immediately at the best available price
- **Behavior**:
  - Buy orders match against the lowest ask price
  - Sell orders match against the highest bid price
  - Partial fills allowed if insufficient liquidity
  - Unfilled portion cancelled (no resting on book)
- **Validation**:
  - Order size >= minimum lot size for the market
  - Order size <= maximum position size allowed
  - User has sufficient margin for the order
- **Execution Priority**: Immediate execution, no queueing

**Limit Orders**
- **Definition**: Orders that execute at a specified price or better
- **Behavior**:
  - Buy orders execute at limit price or lower
  - Sell orders execute at limit price or higher
  - Can rest on order book if not immediately fillable
  - Time-in-force options: GTC (Good Till Cancel), IOC (Immediate or Cancel), FOK (Fill or Kill)
- **Validation**:
  - Price must be valid tick size increment
  - Order size >= minimum lot size
  - Sufficient margin to support the order
- **Order Book Placement**:
  - Price-time priority (FIFO at each price level)
  - Orders ranked by price, then timestamp

**Post-Only Orders**
- **Definition**: Limit orders that only add liquidity (maker orders)
- **Behavior**:
  - Rejected if would immediately match existing orders
  - Always pay maker fees (or receive rebates)
  - Ensures order adds depth to order book
- **Use Case**: Market makers avoiding taker fees
- **Validation**: Same as limit orders + crossing check

#### 2.1.2 Order Matching Engine

**Matching Algorithm**
- **Type**: Price-Time Priority (FIFO)
- **Process**:
  1. Incoming order checked against opposite side of book
  2. Match at best available prices first
  3. Within same price level, match oldest orders first
  4. Continue until order filled or no more matches
  5. Remaining quantity (if limit order) added to book

**Matching Rules**
- **Price Improvement**: Orders can execute at better prices than specified
- **Partial Fills**: 
  - Allowed for all order types
  - Each fill generates separate execution record
  - Position updated after each fill
- **Self-Trade Prevention**:
  - Orders from same account cannot match
  - Incoming order cancelled if would self-trade

**Gas Optimization**
- **Batch Processing**: Multiple orders processed per transaction where possible
- **Order Book Depth**: Limited to top N price levels stored on-chain
- **Lazy Deletion**: Cancelled orders marked rather than removed

#### 2.1.3 Order Book Management

**Book Structure**
- **Bid Side**: Sorted descending by price
- **Ask Side**: Sorted ascending by price
- **Price Levels**: Linked list of orders at each price
- **Order Storage**:
  ```
  Order {
    orderId: uint256
    trader: address
    price: uint256
    size: uint256
    timestamp: uint256
    orderType: enum
    side: enum (buy/sell)
    timeInForce: enum
  }
  ```

**Book Maintenance**
- **Order Lifecycle**:
  1. Validation → Matching → Book Insertion/Execution
  2. Partial fills update order size
  3. Complete fills remove order
  4. Cancellations mark order as inactive
- **Price Level Management**:
  - Empty price levels removed to save gas
  - New levels created as needed
  - Efficient insertion/deletion algorithms

### 2.2 Perpetual Contract Mechanics

#### 2.2.1 Contract Specifications

**Contract Structure**
```
PerpetualMarket {
  symbol: string (e.g., "AAPL-PERP")
  indexPrice: uint256 (from oracle)
  markPrice: uint256 (calculated)
  fundingRate: int256 (positive = longs pay shorts)
  nextFundingTime: uint256
  
  // Market parameters
  tickSize: uint256 (e.g., 0.01 USD)
  lotSize: uint256 (e.g., 0.001 contracts)
  maxLeverage: uint256 (e.g., 100)
  initialMarginRate: uint256 (e.g., 1%)
  maintenanceMarginRate: uint256 (e.g., 0.6%)
  
  // Risk parameters
  maxPositionSize: uint256
  maxOrderSize: uint256
  insuranceFundAddress: address
}
```

**Position Tracking**
```
Position {
  trader: address
  market: address
  size: int256 (positive = long, negative = short)
  entryPrice: uint256
  margin: uint256
  lastFundingIndex: uint256
  realizedPnL: int256
  unrealizedPnL: int256
}
```

#### 2.2.2 Funding Rate Mechanism

**Calculation Formula**
```
Premium = (MarkPrice - IndexPrice) / IndexPrice
FundingRate = Premium * AdjustmentFactor + InterestRate

Where:
- AdjustmentFactor = 3.0 (faster convergence than traditional 1.0)
- InterestRate = 0.0001 (0.01% per funding period)
- FundingRateCap = ±0.01 (1% max per period)
```

**Funding Process**
1. **Calculation Frequency**: Every block, stored for hourly settlement
2. **Settlement**: Every hour on the hour
3. **Payment Flow**:
   - Positive rate: Longs pay shorts
   - Negative rate: Shorts pay longs
   - Payment = Position Size × Mark Price × Funding Rate
4. **Auto-Settlement**: Deducted from/added to position margin

**Premium Index Calculation**
- Sample mark-index spread every minute
- Use time-weighted average over funding period
- Outlier rejection for manipulation resistance

#### 2.2.3 Mark Price Calculation

**Formula**
```
MarkPrice = IndexPrice × (1 + FundingBasis)

Where FundingBasis is the 5-minute EMA of:
(BestBid + BestAsk) / 2 - IndexPrice
---------------------------------
           IndexPrice
```

**Purpose**
- Liquidations based on mark price (not last traded)
- Prevents manipulation through thin order books
- Smooths out temporary price spikes

### 2.3 Margin System

#### 2.3.1 Margin Requirements

**Initial Margin (IM)**
- **Definition**: Minimum collateral to open position
- **Calculation**: 
  ```
  IM = Position Notional Value / Maximum Leverage
  IM = (Position Size × Entry Price) / 100
  ```
- **Application**: Checked on order placement

**Maintenance Margin (MM)**
- **Definition**: Minimum collateral to keep position open
- **Calculation**:
  ```
  MM = Position Notional Value × 0.6%
  MM = Position Size × Mark Price × 0.006
  ```
- **Application**: Continuously monitored for liquidation

**Margin Modes**
- **Isolated Margin** (Launch mode):
  - Margin assigned to specific position
  - Losses limited to position margin
  - No cross-position offsetting
- **Cross Margin** (Future):
  - Shared margin across all positions
  - Unrealized profits offset losses
  - More capital efficient

#### 2.3.2 Leverage Tiers

**Dynamic Leverage Limits**
```
Position Size (Notional) | Max Leverage | IM Rate | MM Rate
< $10,000               | 100x         | 1.0%    | 0.6%
$10,000 - $50,000      | 50x          | 2.0%    | 1.0%
$50,000 - $250,000     | 20x          | 5.0%    | 2.5%
$250,000 - $1,000,000  | 10x          | 10.0%   | 5.0%
> $1,000,000           | 5x           | 20.0%   | 10.0%
```

### 2.4 Liquidation System

#### 2.4.1 Liquidation Triggers

**Trigger Condition**
```
Account Margin Ratio = (Margin + Unrealized PnL) / Maintenance Margin

If Account Margin Ratio < 100%, position is liquidatable
```

**Liquidation Price Calculation**
```
For Long Position:
Liquidation Price = Entry Price × (1 - (IM - MM - Fees) / Position Size)

For Short Position:
Liquidation Price = Entry Price × (1 + (IM - MM - Fees) / Position Size)
```

#### 2.4.2 Liquidation Process

**Liquidation Types**
1. **Partial Liquidation** (Positions > $50,000 notional):
   - Liquidate 50% of position
   - Check if remaining position is safe
   - Repeat if necessary
   - Reduces market impact

2. **Full Liquidation** (Positions ≤ $50,000 notional):
   - Entire position closed at once
   - More gas efficient for small positions

**Liquidation Execution**
1. **Liquidator Selection**:
   - Priority given to whitelisted liquidators (first 100ms)
   - Open to all liquidators after priority window
   - Liquidators compete on speed

2. **Order Placement**:
   - Liquidation order placed as IOC market order
   - Executes against existing order book
   - Slippage absorbed by liquidated trader

3. **Penalty Distribution**:
   ```
   Liquidation Penalty = Position Value × 1.5%
   
   Distribution:
   - Liquidator Reward: 0.5%
   - Insurance Fund: 1.0%
   ```

#### 2.4.3 Insurance Fund

**Purpose**
- Cover bad debt when liquidation price worse than bankruptcy price
- Maintain system solvency
- Prevent socialized losses

**Funding Sources**
- Liquidation penalties (1% of liquidated positions)
- Trading fees allocation (40% of all fees)
- Direct protocol funding (initial seed)

**Usage Rules**
```
If (Liquidation Proceeds < Outstanding Margin Debt):
    Deficit = Outstanding Debt - Liquidation Proceeds
    Insurance Fund covers Deficit
    
If (Insurance Fund Depleted):
    Initiate Auto-Deleveraging (ADL)
```

**Auto-Deleveraging (ADL)**
- Last resort when insurance fund insufficient
- Profitable traders on opposite side closed
- Priority: Highest profit ratio + leverage
- Affected traders notified immediately

### 2.5 Collateral Management

#### 2.5.1 Accepted Collateral Types

**Stablecoin Support**
```
Collateral {
  USDC: {
    address: 0x...,
    weight: 100%,    // No haircut
    decimals: 6,
    priceFeed: Chainlink USDC/USD
  },
  USDT: {
    address: 0x...,
    weight: 98%,     // 2% haircut
    decimals: 6,
    priceFeed: Chainlink USDT/USD
  },
  DAI: {
    address: 0x...,
    weight: 95%,     // 5% haircut
    decimals: 18,
    priceFeed: Chainlink DAI/USD
  }
}
```

**Collateral Valuation**
```
Collateral Value = Σ(Token Balance × Token Price × Weight)
```

#### 2.5.2 Deposit/Withdrawal

**Deposit Process**
1. User approves token transfer
2. Contract pulls tokens from user
3. Credits user's margin account
4. Updates available margin for trading

**Withdrawal Rules**
- Only excess margin can be withdrawn
- Must maintain margin requirements
- Calculation:
  ```
  Withdrawable = Total Collateral - Initial Margin for All Positions
  ```
- Withdrawals blocked during liquidation

### 2.6 Oracle System

#### 2.6.1 Multi-Oracle Architecture

**Oracle Interface**
```
interface IPriceOracle {
  function getPrice(symbol) returns (price, confidence, timestamp);
  function subscribe(symbol);
  function updatePrice(symbol, price, confidence);
}
```

**Supported Oracles**
1. **Primary**: High-frequency specialized oracles
2. **Secondary**: Chainlink price feeds
3. **Tertiary**: Other DeFi oracles (backup)

#### 2.6.2 Price Aggregation

**Aggregation Algorithm**
1. **Collect Prices**: Gather from all active oracles
2. **Outlier Detection**:
   ```
   Median = median(all prices)
   MAD = median(|price - Median|)
   Reject if: |price - Median| > 3 × MAD
   ```
3. **Weighted Average**:
   ```
   Weight based on:
   - Oracle reputation score (0-100)
   - Recency of update (exponential decay)
   - Confidence interval provided
   ```
4. **Confidence Calculation**:
   ```
   System Confidence = Min(Oracle Confidences) × Agreement Factor
   Where Agreement Factor = 1 - (StdDev / Mean)
   ```

**Oracle Monitoring**
- Track update frequency
- Monitor price deviations
- Calculate oracle reliability scores
- Auto-disable consistently poor oracles

### 2.7 Risk Management

#### 2.7.1 Position Limits

**Global Limits Per Market**
```
Market Risk Parameters {
  maxOpenInterest: uint256,        // e.g., $100M notional
  maxPositionSize: uint256,        // e.g., $5M per trader
  maxOrderSize: uint256,           // e.g., $1M per order
  concentrationLimit: uint256      // e.g., 20% of OI per trader
}
```

**Enforcement**
- Check on order placement
- Prevent market manipulation
- Ensure liquidation feasibility

#### 2.7.2 Circuit Breakers

**Price Movement Limits**
```
Short-term (5 min): ±5% from reference price
Medium-term (1 hour): ±10% from reference price
Daily: ±20% from previous day close
```

**Actions on Trigger**
1. Temporary trading halt (5 minutes)
2. Cancel all orders
3. Widen spread requirements
4. Reduce maximum leverage
5. Resume with restrictions

#### 2.7.3 Emergency Controls

**Admin Functions** (Timelocked)
- Pause new position opening
- Force-settle all positions at oracle price
- Adjust risk parameters
- Add/remove oracle sources
- Update fee structure

**Governance Controls**
- 48-hour timelock on critical changes
- Multi-sig required for emergency actions
- Transparent on-chain execution

## 3. User Workflows

### 3.1 Trader Workflows

#### 3.1.1 Opening a Position

**Flow: Market Buy Order**
1. **Check Requirements**
   - User has sufficient collateral
   - Position size within limits
   - Market is active

2. **Calculate Required Margin**
   ```
   Order Size = 1 BTC-PERP
   Mark Price = $50,000
   Leverage = 50x
   Required Margin = $50,000 × 1 / 50 = $1,000 USDC
   ```

3. **Execute Order**
   - Deduct margin from available balance
   - Match against order book
   - Create/update position record
   - Emit PositionOpened event

4. **Post-Execution**
   - Update user's margin balance
   - Calculate new liquidation price
   - Start tracking funding payments

**Flow: Limit Sell Order**
1. **Validation**
   - Price is valid tick increment
   - Size meets minimum lot size
   - Margin sufficient for max loss

2. **Order Placement**
   - Add to order book at specified price
   - Reserve margin for potential position
   - Emit OrderPlaced event

3. **Execution (when matched)**
   - Update position (size becomes negative)
   - Realize any PnL from position reduction
   - Update margin requirements

#### 3.1.2 Closing a Position

**Full Position Close**
1. **Calculate Current P&L**
   ```
   Entry Price: $50,000
   Current Price: $52,000
   Position: 1 BTC Long
   P&L = (52,000 - 50,000) × 1 = $2,000
   Fees = 52,000 × 1 × 0.05% = $26
   Net P&L = $2,000 - $26 = $1,974
   ```

2. **Execute Close**
   - Place market order opposite to position
   - Match against order book
   - Calculate final P&L including fees

3. **Settlement**
   - Add P&L to margin balance
   - Release remaining margin
   - Delete position record
   - Emit PositionClosed event

**Partial Position Close**
- Similar flow but size specified
- Position record updated, not deleted
- Proportional margin released

#### 3.1.3 Managing Margin

**Adding Margin**
1. Approve collateral token
2. Call `addMargin(market, amount)`
3. Margin added to position
4. Liquidation price improved
5. Emit MarginAdded event

**Removing Margin**
1. Calculate max removable
2. Call `removeMargin(market, amount)`
3. Check position still safe
4. Transfer tokens to trader
5. Emit MarginRemoved event

### 3.2 Liquidator Workflows

#### 3.2.1 Monitoring Positions

**Continuous Monitoring**
```
For each position:
  marginRatio = (margin + unrealizedPnL) / maintenanceMargin
  if (marginRatio < 100%):
    position is liquidatable
```

**Liquidation Opportunity Detection**
1. Monitor PositionUpdated events
2. Calculate margin ratios off-chain
3. Identify profitable liquidations
4. Account for gas costs and rewards

#### 3.2.2 Executing Liquidations

**Liquidation Execution**
1. **Call Liquidation Function**
   ```
   liquidatePosition(trader, market)
   ```

2. **Contract Validates**
   - Position truly liquidatable
   - Liquidator has priority (if applicable)
   - Calculate liquidation size

3. **Execute Liquidation**
   - Create liquidation order
   - Match against order book
   - Calculate penalties and rewards

4. **Settlement**
   - Pay liquidator reward (0.5%)
   - Send penalty to insurance (1.0%)
   - Update/close position
   - Emit PositionLiquidated event

### 3.3 Market Maker Workflows

#### 3.3.1 Providing Liquidity

**Continuous Quoting**
1. **Calculate Fair Price**
   - Use oracle price as reference
   - Add/subtract spread

2. **Place Orders**
   ```
   Bid: Oracle Price - Spread
   Ask: Oracle Price + Spread
   Size: Based on risk limits
   ```

3. **Use Post-Only Orders**
   - Ensure maker status
   - Avoid taker fees
   - Orders rejected if would cross

4. **Manage Inventory**
   - Track position exposure
   - Adjust quotes based on inventory
   - Hedge in spot markets if needed

#### 3.3.2 Managing Orders

**Order Updates**
1. **Cancel Stale Orders**
   - Monitor price movements
   - Cancel if too far from market
   - Replace with new orders

2. **Adjust for Funding**
   - Consider funding rate in pricing
   - Bias quotes based on funding direction
   - Capture funding arbitrage

3. **Risk Management**
   - Set position limits
   - Monitor margin usage
   - Implement stop-loss logic

## 4. Technical Architecture

### 4.1 Smart Contract Architecture

#### 4.1.1 Contract Structure

**Core Contracts**
```
PerpetualExchange.sol
├── OrderBook.sol
├── PositionManager.sol
├── MarginManager.sol
├── LiquidationEngine.sol
├── FundingRateCalculator.sol
├── OracleAggregator.sol
├── InsuranceFund.sol
└── MarketRegistry.sol
```

**Relationships**
- PerpetualExchange: Main entry point
- OrderBook: Per-market order management
- PositionManager: Position tracking and P&L
- MarginManager: Collateral and margin logic
- LiquidationEngine: Liquidation execution
- FundingRateCalculator: Funding calculations
- OracleAggregator: Price feed management
- InsuranceFund: Bad debt coverage
- MarketRegistry: Market configuration

#### 4.1.2 Key Interfaces

**IPerpetualExchange**
```solidity
interface IPerpetualExchange {
  // Trading
  function placeOrder(market, side, orderType, price, size) external;
  function cancelOrder(orderId) external;
  
  // Position Management
  function closePosition(market) external;
  function addMargin(market, amount) external;
  function removeMargin(market, amount) external;
  
  // Views
  function getPosition(trader, market) external view returns (Position);
  function getOrderbook(market, depth) external view returns (OrderbookData);
}
```

**IOrderBook**
```solidity
interface IOrderBook {
  function addOrder(order) external returns (orderId);
  function cancelOrder(orderId) external;
  function matchOrders() external returns (fills);
  function getBestBid() external view returns (price, size);
  function getBestAsk() external view returns (price, size);
}
```

### 4.2 Data Storage Optimization

#### 4.2.1 Storage Patterns

**Packed Structs**
```solidity
// Efficient position storage - fits in 3 slots
struct Position {
  int128 size;           // 16 bytes
  uint128 margin;        // 16 bytes
  uint128 entryPrice;    // 16 bytes  
  uint64 lastFundingIdx; // 8 bytes
  uint64 openTimestamp;  // 8 bytes
}
```

**Mapping Structures**
```solidity
// Nested mappings for positions
mapping(address => mapping(address => Position)) positions;
// trader => market => position

// Order storage
mapping(uint256 => Order) orders;
mapping(address => uint256[]) userOrders;
```

#### 4.2.2 Gas Optimizations

**Batch Operations**
- Process multiple orders per transaction
- Aggregate position updates
- Batch funding settlements

**Storage Minimization**
- Use events for historical data
- Store only active state on-chain
- Implement efficient data structures

## 5. Data Models

### 5.1 Core Entities

#### 5.1.1 Market Configuration
```
Market {
  symbol: string
  baseAsset: string (e.g., "AAPL")
  isActive: bool
  
  // Trading parameters
  tickSize: uint256
  lotSize: uint256
  maxLeverage: uint256
  
  // Risk parameters  
  initialMarginRate: uint256
  maintenanceMarginRate: uint256
  maxPositionSize: uint256
  maxOrderSize: uint256
  
  // Oracle configuration
  oracleIds: address[]
  oracleWeights: uint256[]
  
  // Fee structure
  makerFee: int256 (negative = rebate)
  takerFee: uint256
  
  // Statistics
  openInterest: uint256
  volume24h: uint256
  nextFundingTime: uint256
}
```

#### 5.1.2 Order Structure
```
Order {
  orderId: uint256
  trader: address
  market: address
  
  // Order details
  side: enum (BUY, SELL)
  orderType: enum (MARKET, LIMIT, POST_ONLY)
  price: uint256
  size: uint256
  filled: uint256
  
  // Status
  status: enum (ACTIVE, FILLED, CANCELLED)
  timestamp: uint256
  timeInForce: enum (GTC, IOC, FOK)
  
  // Execution
  avgFillPrice: uint256
  fees: uint256
}
```

#### 5.1.3 Position Structure
```
Position {
  trader: address
  market: address
  
  // Position details
  size: int256 (positive=long, negative=short)
  margin: uint256
  entryPrice: uint256
  markPrice: uint256
  
  // P&L tracking
  realizedPnL: int256
  unrealizedPnL: int256
  fundingPayments: int256
  
  // Risk metrics
  liquidationPrice: uint256
  marginRatio: uint256
  leverage: uint256
  
  // Timestamps
  openTime: uint256
  lastUpdateTime: uint256
}
```

### 5.2 Event Schema

#### 5.2.1 Trading Events
```
event OrderPlaced(
  address indexed trader,
  address indexed market,
  uint256 orderId,
  uint256 price,
  uint256 size,
  OrderSide side,
  OrderType orderType
);

event OrderCancelled(
  uint256 indexed orderId,
  address indexed trader
);

event OrderFilled(
  uint256 indexed orderId,
  uint256 fillPrice,
  uint256 fillSize,
  uint256 fee
);

event PositionOpened(
  address indexed trader,
  address indexed market,
  int256 size,
  uint256 entryPrice,
  uint256 margin
);

event PositionClosed(
  address indexed trader,
  address indexed market,
  uint256 closePrice,
  int256 realizedPnL
);
```

#### 5.2.2 Risk Events
```
event PositionLiquidated(
  address indexed trader,
  address indexed market,
  address indexed liquidator,
  uint256 size,
  uint256 liquidationPrice,
  uint256 penalty
);

event MarginAdded(
  address indexed trader,
  address indexed market,
  uint256 amount
);

event FundingPaid(
  address indexed market,
  int256 fundingRate,
  uint256 timestamp
);
```

## 6. Non-Functional Requirements

### 6.1 Performance Requirements

#### 6.1.1 Throughput
- **Order Processing**: 1000+ orders per second
- **Position Updates**: 10,000+ per second
- **Price Updates**: Sub-second latency
- **Liquidation Detection**: <1 second delay

#### 6.1.2 Scalability
- **Markets**: Support 1000+ perpetual markets
- **Users**: Handle 1M+ active traders
- **Orders**: 10M+ open orders across all markets
- **Positions**: 5M+ open positions

### 6.2 Security Requirements

#### 6.2.1 Smart Contract Security
- **Audits**: 3+ independent security audits
- **Formal Verification**: Critical functions verified
- **Bug Bounty**: Ongoing program with up to $1M rewards
- **Upgrade Mechanism**: Timelocked proxy pattern

#### 6.2.2 Oracle Security
- **Multiple Sources**: Minimum 3 oracles per asset
- **Manipulation Resistance**: Outlier detection
- **Failover**: Automatic oracle switching
- **Circuit Breakers**: Halt on suspicious prices

### 6.3 Reliability Requirements

#### 6.3.1 Availability
- **Uptime Target**: 99.9% (8.76 hours downtime/year)
- **RPC Redundancy**: Multiple node providers
- **Fallback Mechanisms**: Graceful degradation

#### 6.3.2 Data Integrity
- **State Consistency**: ACID properties maintained
- **Reconciliation**: Regular state verification
- **Backup**: Event log enables full reconstruction

## 7. Implementation Guidelines

### 7.1 Development Phases

#### Phase 1: Core Infrastructure (Months 1-2)
1. Smart contract architecture
2. Basic order book implementation
3. Position management system
4. Margin calculations
5. Initial testing framework

#### Phase 2: Trading Features (Months 2-3)
1. Market and limit orders
2. Funding rate mechanism
3. Basic liquidation engine
4. Oracle integration
5. Integration testing

#### Phase 3: Risk Management (Months 3-4)
1. Advanced liquidation logic
2. Insurance fund
3. Circuit breakers
4. Position limits
5. Security testing

#### Phase 4: Advanced Features (Months 4-5)
1. Post-only orders
2. Multi-collateral support
3. Performance optimizations
4. Monitoring systems
5. Mainnet preparation

#### Phase 5: Launch Preparation (Month 6)
1. Security audits
2. Bug fixes
3. Documentation
4. Deployment scripts
5. Mainnet launch

### 7.2 Testing Strategy

#### 7.2.1 Unit Testing
- 100% code coverage target
- Test all edge cases
- Gas consumption tests
- Fuzzing for unexpected inputs

#### 7.2.2 Integration Testing
- Multi-contract interactions
- End-to-end workflows
- Performance benchmarks
- Stress testing

#### 7.2.3 Security Testing
- Reentrancy checks
- Integer overflow/underflow
- Access control verification
- Economic attack simulations

### 7.3 Deployment Strategy

#### 7.3.1 Testnet Deployment
1. Deploy to testnet (Goerli/Mumbai)
2. Public testing period (1 month)
3. Bug bounty program
4. Performance optimization

#### 7.3.2 Mainnet Deployment
1. Gradual rollout
2. Limited markets initially
3. Conservative risk parameters
4. Progressive decentralization

## 8. Appendices

### 8.1 Glossary

- **ADL**: Auto-deleveraging, forced position closure when insurance fund depleted
- **Funding Rate**: Periodic payment between longs and shorts
- **Initial Margin**: Minimum collateral to open position
- **Liquidation**: Forced position closure due to insufficient margin
- **Maintenance Margin**: Minimum collateral to keep position open
- **Mark Price**: Reference price for liquidations and P&L
- **Notional Value**: Position size × Price
- **Oracle**: External price data provider
- **P&L**: Profit and Loss
- **Slippage**: Difference between expected and actual execution price

### 8.2 Example Calculations

#### Opening a Long Position
```
Market: BTC-PERP
Entry Price: $50,000
Position Size: 2 BTC
Leverage: 50x

Initial Margin = (2 × $50,000) / 50 = $2,000
Maintenance Margin = 2 × $50,000 × 0.006 = $600
Liquidation Price = $50,000 × (1 - (0.02 - 0.006) / 2) = $49,650
```

#### Funding Payment
```
Position: 5 ETH-PERP Long
Mark Price: $3,000
Funding Rate: 0.01% (longs pay shorts)

Funding Payment = 5 × $3,000 × 0.0001 = $1.50 (paid to shorts)
```

### 8.3 Risk Parameters by Asset Class

```
Asset Class     | Max Leverage | Init Margin | Maint Margin
Blue-chip Stocks| 50x         | 2%          | 1%
Tech Stocks     | 20x         | 5%          | 2.5%
Small-cap       | 10x         | 10%         | 5%
Index ETFs      | 100x        | 1%          | 0.6%
```

---

This PRD provides comprehensive specifications for implementing the DEX Perpetual Contracts Protocol. All features are designed with retail trader accessibility in mind while maintaining the sophistication required for a high-performance derivatives exchange.