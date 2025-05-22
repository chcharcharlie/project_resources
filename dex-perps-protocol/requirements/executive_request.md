# Executive Requirements Document - DEX Perpetual Contracts Exchange

## Product Overview

You are building a decentralized perpetual contracts exchange that enables traders to speculate on US stock prices through on-chain perpetual futures contracts. The platform will support 100X leverage, multiple collateral types (primarily stablecoins), and will handle all core trading functions through smart contracts without any frontend UI.

## Business Objectives

1. **Create a High-Performance DEX**: Target $5B daily trading volume comparable to leading DEXs
2. **Enable Stock Market Access**: Provide crypto users exposure to US equity markets through perpetual contracts
3. **Maintain Decentralization**: All core functions executed on-chain via smart contracts
4. **Support Professional Trading**: Offer advanced features and high leverage for sophisticated traders

## Target Users

- **Primary**: Professional crypto traders seeking leveraged exposure to US equity markets
- **Secondary**: DeFi protocols and market makers providing liquidity
- **Tertiary**: Arbitrageurs maintaining price efficiency between markets

## Core Functionality

### 1. Order Book Management
- **Implementation**: Fully on-chain order book (not AMM-based)
- **Order Types**: 
  - Start with limit and market orders
  - Future: stop-loss, take-profit, and other advanced order types
- **Matching Algorithm**: Price-time priority (FIFO)
- **Post-Only Orders**: Include at launch to allow makers to ensure orders add liquidity
- **Iceberg Orders**: Defer to post-launch (allows hiding large order sizes)

### 2. Perpetual Contract Mechanics
- **Funding Rate**:
  - Calculated and settled every 1 hour
  - Modified traditional formula with faster premium adjustments to reduce imbalanced market risk
  - Include funding rate cap mechanism to prevent extreme rates
- **Position Management**:
  - Start with one-way mode (simpler implementation)
  - Future consideration for hedge mode (allowing simultaneous long/short)
  - Global position limits per market initially

### 3. Margin and Leverage System
- **Maximum Leverage**: 100X
- **Initial Margin**: 1% (for 100X leverage)
- **Maintenance Margin**: 0.6% (following market norms for high leverage)
- **Margin Mode**: Start with isolated margin per position
- **Portfolio Margining**: Consider for future implementation

### 4. Liquidation Engine
- **Mechanism**: Hybrid approach
  - Partial liquidations for larger positions to reduce market impact
  - Full liquidations for smaller positions
- **Liquidator Access**:
  - Prioritized permissioned liquidators for efficiency
  - Permissionless fallback to ensure system security
- **Liquidation Penalty**: 1.5% (market standard)
- **Insurance Fund**: Required to cover bad debt from liquidations

### 5. Price Oracle System
- **Multi-Oracle Support**: Design interfaces for multiple oracle providers
- **Price Aggregation**:
  - Outlier rejection algorithm
  - Weighted average of remaining prices
  - Performance tracking for each oracle
- **Oracle Interface Requirements**:
  - Support high-frequency updates (sub-second ideally)
  - Include confidence intervals/bands
  - Fallback mechanisms for oracle failures

### 6. Collateral Management
- **Accepted Collateral**:
  - Primary: USDC, USDT, DAI
  - Future: Other stablecoins based on liquidity
- **Collateral Weights**: Different haircuts for different stablecoins based on risk
- **Cross-Collateral**: Not at launch (keep it simple initially)

### 7. Risk Management
- **Insurance Fund**:
  - Funded by portion of trading fees and liquidation penalties
  - Automatic bad debt coverage
- **Circuit Breakers**: Consider emergency pause mechanisms
- **Position Limits**: Global limits per market to manage systemic risk

## Technical Requirements

### Platform Selection
- **Target Chains**: 
  - Layer 2s: Polygon, Arbitrum, Optimism (lower fees, good DeFi ecosystem)
  - Layer 1: Solana (highest throughput but different architecture)
- **Performance Requirements**:
  - Handle 1000+ orders per second
  - Sub-second order matching (within block time constraints)
  - Gas optimization critical for on-chain order book

### Smart Contract Architecture
- **Modular Design**: Separate contracts for:
  - Order book management
  - Position tracking
  - Margin calculations
  - Liquidation engine
  - Oracle aggregation
  - Insurance fund
- **Upgradeability**: Proxy pattern for critical components
- **Security**: Multiple audits required before mainnet

### Market Creation Parameters
- **Per Market Configuration**:
  - Tick Size: Minimum price increment (e.g., 0.01 for most stocks)
  - Lot Size: Minimum order size
  - Max Leverage: Can vary by asset volatility
  - Initial/Maintenance Margin: Asset-specific based on volatility
  - Funding Rate Parameters: Premium/discount thresholds
  - Oracle Sources: Which oracles to use for each market
  - Trading Hours: 24/7 or match traditional market hours

## Constraints

### Technical Constraints
- **Block Time Limitations**: Order matching speed limited by chain selection
- **Gas Costs**: On-chain order book requires significant optimization
- **Oracle Latency**: Price updates limited by oracle infrastructure

### Business Constraints
- **No Frontend**: Smart contracts only, third parties can build UIs
- **Permissioned Market Creation**: Only approved entities can create new markets
- **Regulatory**: No KYC/AML requirements specified

## Success Metrics

1. **Trading Volume**: Achieve $5B daily volume within 12 months
2. **Market Efficiency**: Perpetual prices track spot prices within 0.1%
3. **System Reliability**: 99.9% uptime with no loss of funds
4. **Liquidation Efficiency**: <1% bad debt ratio
5. **Gas Efficiency**: Competitive transaction costs vs other DEXs

## Fee Structure

Based on market analysis:
- **Maker Fees**: 0.02% (potentially negative/rebates for liquidity providers)
- **Taker Fees**: 0.05%
- **Fee Distribution**:
  - Insurance Fund: 40%
  - Protocol Treasury: 40%
  - Future: Liquidity Provider Rewards: 20%

## Future Considerations

1. **Advanced Order Types**: Stop-loss, take-profit, trailing stops, iceberg orders
2. **Hedge Mode**: Allow simultaneous long/short positions
3. **Portfolio Margining**: Cross-margining between positions
4. **Additional Collateral**: ETH, WBTC, other assets
5. **Governance Token**: For parameter updates and protocol governance
6. **Mobile/API SDKs**: Enable third-party integrations

## Open Questions Requiring Further Discussion

1. **Oracle Failure Handling**: Specific mechanisms for when oracles fail or provide stale data
2. **Maximum Acceptable Price Delay**: Latency tolerance for price updates
3. **Maker/Taker Fee Differentiation**: Exact fee tiers and volume discounts
4. **Emergency Procedures**: Admin controls for extreme market conditions
5. **Liquidation Monitoring**: Confirm if monitoring should be on-chain or off-chain

## Next Steps

1. Finalize chain selection based on technical requirements
2. Design detailed smart contract architecture
3. Specify exact oracle integration requirements
4. Define initial market parameters for launch assets
5. Create security and audit plan

---

This document represents the confirmed requirements for the DEX perpetual contracts exchange based on user specifications and market research. It will serve as the foundation for the Product Requirements Document (PRD) in the next phase.