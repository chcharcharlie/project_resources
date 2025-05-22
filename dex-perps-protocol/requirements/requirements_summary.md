# Requirements Gathering Summary

## Phase Completion Date: 2025-01-22

## Key Requirements Gathered

### 1. Product Vision
- Decentralized perpetual contracts exchange for US stock prices
- Smart contracts only (no UI)
- Target $5B daily trading volume
- Support 100X leverage

### 2. Technical Decisions
- **Order Book**: Fully on-chain (not AMM)
- **Collateral**: Multiple stablecoins (USDC, USDT, DAI primarily)
- **Chain**: TBD between L2s (Polygon, Arbitrum) or Solana
- **Oracle**: Multi-oracle support with aggregation

### 3. Market Research Findings

#### Position Management
- Most exchanges offer both one-way and hedge modes
- One-way mode is simpler to implement
- Hedge mode allows simultaneous long/short positions

#### Order Types
- **Post-only orders**: Recommended for launch to ensure makers add liquidity
- **Iceberg orders**: Complex feature, defer to post-launch

#### Margin Requirements (Market Norms)
- For 20X leverage: Initial 5%, Maintenance 3%
- For 100X leverage: Initial 1%, Maintenance 0.5-0.6%
- Our choice: 1% initial, 0.6% maintenance

#### Liquidation Mechanisms
- Industry uses both partial and full liquidations
- Partial liquidations reduce market impact for large positions
- Standard liquidation penalty: 1-1.5%

#### Trading Fees (Market Analysis)
- Maker fees: 0.02% (some offer rebates)
- Taker fees: 0.04-0.07%
- dYdX and Binance are on the lower end

#### Market Creation Parameters Needed
- Tick size (minimum price increment)
- Lot size (minimum order quantity)
- Leverage limits per asset
- Margin requirements per asset
- Oracle configuration
- Trading hours (24/7 vs traditional)

### 4. Design Choices Made
- Start with one-way position mode
- Funding rate every 1 hour with modified formula
- Permissioned liquidators with permissionless fallback
- Insurance fund funded by fees and liquidation penalties
- Permissioned market creation

### 5. Features Deferred
- Cross-collateral support
- Portfolio margining
- Advanced order types (stop-loss, take-profit)
- Iceberg orders
- Non-stablecoin collateral
- Governance token

## Next Steps
The executive requirements document has been completed and confirmed. The project is ready to move to the Product Manager phase to create a detailed PRD based on these requirements.