# Core Protocol Engine Specification

## Overview
The Core Protocol Engine manages positions, margins, funding rates, and P&L calculations. This is the largest and most critical module, handling all position lifecycle operations and financial calculations.

## Component Structure

```
CoreProtocol/
├── contracts/
│   ├── PositionManager.sol      // Position lifecycle management
│   ├── MarginManager.sol        // Margin calculations and management
│   ├── FundingRateEngine.sol    // Funding rate calculations
│   ├── PnLCalculator.sol        // Profit/Loss calculations
│   └── CollateralManager.sol    // Multi-collateral support
├── interfaces/
│   ├── ICoreProtocol.sol        // Main interface
│   └── IPositionManager.sol     // Position interface
└── libraries/
    ├── PositionLib.sol          // Position utilities
    ├── MathLib.sol              // Fixed-point math
    └── FundingLib.sol           // Funding calculations
```

## Technical Specifications

### Data Structures

```solidity
// Position representation
struct Position {
    // Core fields (Slot 1)
    int128 size;              // Positive = long, negative = short
    uint128 margin;           // Deposited margin in USD value
    
    // Price data (Slot 2)
    uint128 avgEntryPrice;    // Volume-weighted average entry
    uint64 lastFundingIndex;  // For funding settlement
    uint64 openTimestamp;     // Position open time
    
    // P&L tracking (Slot 3)
    int128 realizedPnL;       // Lifetime realized P&L
    uint128 marginUsed;       // Margin locked for orders
}

// Account representation
struct Account {
    // Collateral balances
    mapping(address => uint256) collateralBalances;
    
    // Position keys
    EnumerableSet.Bytes32Set positionKeys;
    
    // Account metrics
    uint256 totalMarginUsed;
    int256 totalUnrealizedPnL;
    int256 totalRealizedPnL;
}

// Funding rate data
struct FundingData {
    int256 currentRate;       // Current funding rate (per hour)
    uint256 lastUpdateTime;   // Last calculation time
    uint256 cumulativeIndex;  // For position settlement
    
    // Premium calculation
    uint256 premiumEMA;       // Exponential moving average
    uint256 lastMarkPrice;    // Last mark price
    uint256 lastIndexPrice;   // Last index price
}

// Market aggregated data
struct MarketData {
    uint256 longOpenInterest;
    uint256 shortOpenInterest;
    uint256 maxOpenInterest;
    FundingData funding;
}
```

### Core Functions

#### Position Opening
```solidity
function openPosition(
    address trader,
    address market,
    int256 sizeDelta,
    uint256 executionPrice,
    uint256 marginDelta
) external onlyTradingEngine returns (bytes32 positionKey) {
    // 1. Validate inputs
    require(sizeDelta != 0, "Invalid size");
    MarketConfig memory config = marketRegistry.getMarketConfig(market);
    
    // 2. Calculate position key
    positionKey = keccak256(abi.encodePacked(trader, market));
    Position storage position = positions[positionKey];
    
    // 3. Check if increasing or decreasing position
    bool isNewPosition = position.size == 0;
    bool isIncreasing = (sizeDelta > 0) == (position.size >= 0);
    
    // 4. Handle position decrease (realize P&L)
    if (!isIncreasing && position.size != 0) {
        int256 realizedPnL = _realizePnL(position, sizeDelta, executionPrice);
        position.realizedPnL += realizedPnL;
        accounts[trader].totalRealizedPnL += realizedPnL;
    }
    
    // 5. Update position size and entry price
    int256 newSize = position.size + sizeDelta;
    if (isIncreasing) {
        position.avgEntryPrice = _calculateAvgEntryPrice(
            position.size,
            position.avgEntryPrice,
            sizeDelta,
            executionPrice
        );
    }
    position.size = newSize;
    
    // 6. Handle margin
    if (marginDelta > 0) {
        position.margin += marginDelta;
        _transferMarginIn(trader, marginDelta);
    }
    
    // 7. Settle funding before position change
    _settleFunding(trader, market, position);
    
    // 8. Validate position
    _validatePosition(position, config);
    
    // 9. Update market data
    _updateOpenInterest(market, sizeDelta);
    
    // 10. Emit events
    emit PositionUpdated(
        trader,
        market,
        position.size,
        position.avgEntryPrice,
        position.margin,
        position.realizedPnL
    );
}
```

#### Margin Management
```solidity
function addMargin(
    address market,
    uint256 amount,
    address collateralToken
) external {
    // 1. Validate collateral token
    CollateralConfig memory config = collateralConfigs[collateralToken];
    require(config.isEnabled, "Invalid collateral");
    
    // 2. Transfer collateral
    IERC20(collateralToken).safeTransferFrom(msg.sender, address(this), amount);
    
    // 3. Calculate USD value with haircut
    uint256 usdValue = _getCollateralValue(collateralToken, amount, config.weight);
    
    // 4. Update position margin
    bytes32 positionKey = keccak256(abi.encodePacked(msg.sender, market));
    Position storage position = positions[positionKey];
    require(position.size != 0, "No position");
    
    position.margin += usdValue;
    accounts[msg.sender].collateralBalances[collateralToken] += amount;
    
    // 5. Emit event
    emit MarginAdded(msg.sender, market, usdValue, collateralToken, amount);
}

function removeMargin(
    address market,
    uint256 amount
) external {
    // 1. Get position
    bytes32 positionKey = keccak256(abi.encodePacked(msg.sender, market));
    Position storage position = positions[positionKey];
    require(position.size != 0, "No position");
    
    // 2. Check margin requirements after removal
    uint256 newMargin = position.margin - amount;
    uint256 requiredMargin = _calculateRequiredMargin(
        position.size,
        marketRegistry.getMarkPrice(market),
        marketRegistry.getMarketConfig(market)
    );
    require(newMargin >= requiredMargin, "Insufficient margin");
    
    // 3. Update position
    position.margin = newMargin;
    
    // 4. Transfer collateral back
    _transferMarginOut(msg.sender, amount);
    
    // 5. Emit event
    emit MarginRemoved(msg.sender, market, amount);
}
```

#### Funding Rate System
```solidity
function updateFundingRate(address market) external {
    MarketData storage marketData = markets[market];
    FundingData storage funding = marketData.funding;
    
    // 1. Check if update needed (every minute)
    if (block.timestamp < funding.lastUpdateTime + 60) {
        return;
    }
    
    // 2. Get current prices
    uint256 markPrice = oracle.getMarkPrice(market);
    uint256 indexPrice = oracle.getIndexPrice(market);
    
    // 3. Calculate premium with 3x adjustment factor
    int256 premium = int256(markPrice) - int256(indexPrice);
    int256 premiumRate = (premium * 3 * PRECISION) / int256(indexPrice);
    
    // 4. Update EMA of premium
    uint256 alpha = PRECISION / 10; // 0.1 smoothing factor
    funding.premiumEMA = uint256(
        (int256(funding.premiumEMA) * (int256(PRECISION) - int256(alpha)) +
         premiumRate * int256(alpha)) / int256(PRECISION)
    );
    
    // 5. Calculate funding rate (capped at ±1%)
    int256 interestRate = 10; // 0.01% in basis points
    int256 fundingRate = int256(funding.premiumEMA) + interestRate;
    fundingRate = _clamp(fundingRate, -10000, 10000); // ±1% cap
    
    // 6. Update funding data
    funding.currentRate = fundingRate;
    funding.lastUpdateTime = block.timestamp;
    funding.lastMarkPrice = markPrice;
    funding.lastIndexPrice = indexPrice;
    
    emit FundingRateUpdated(market, fundingRate, markPrice, indexPrice);
}

function settleFunding(address trader, address market) public {
    bytes32 positionKey = keccak256(abi.encodePacked(trader, market));
    Position storage position = positions[positionKey];
    
    if (position.size == 0) return;
    
    _settleFunding(trader, market, position);
}

function _settleFunding(
    address trader,
    address market,
    Position storage position
) internal {
    FundingData storage funding = markets[market].funding;
    
    // 1. Calculate funding since last settlement
    uint256 fundingDelta = funding.cumulativeIndex - position.lastFundingIndex;
    if (fundingDelta == 0) return;
    
    // 2. Calculate funding payment
    // Positive rate: longs pay shorts
    // Negative rate: shorts pay longs
    int256 fundingPayment = (position.size * int256(fundingDelta)) / int256(PRECISION);
    
    // 3. Apply to position
    position.margin = uint256(int256(position.margin) - fundingPayment);
    position.lastFundingIndex = funding.cumulativeIndex;
    
    // 4. Emit event
    emit FundingPaid(trader, market, fundingPayment, funding.currentRate);
}
```

#### P&L Calculations
```solidity
function getPositionPnL(address trader, address market)
    external
    view
    returns (
        int256 unrealizedPnL,
        int256 realizedPnL,
        uint256 marginRatio
    )
{
    bytes32 positionKey = keccak256(abi.encodePacked(trader, market));
    Position memory position = positions[positionKey];
    
    if (position.size == 0) {
        return (0, position.realizedPnL, 0);
    }
    
    uint256 markPrice = oracle.getMarkPrice(market);
    
    // Calculate unrealized P&L
    unrealizedPnL = _calculateUnrealizedPnL(position, markPrice);
    realizedPnL = position.realizedPnL;
    
    // Calculate margin ratio
    uint256 positionValue = _abs(position.size) * markPrice / PRECISION;
    uint256 effectiveMargin = uint256(int256(position.margin) + unrealizedPnL);
    marginRatio = (effectiveMargin * PRECISION) / positionValue;
}

function _calculateUnrealizedPnL(
    Position memory position,
    uint256 currentPrice
) internal pure returns (int256) {
    if (position.size == 0) return 0;
    
    // P&L = (currentPrice - avgEntryPrice) * size
    int256 priceDelta = int256(currentPrice) - int256(position.avgEntryPrice);
    return (priceDelta * position.size) / int256(PRECISION);
}

function _realizePnL(
    Position memory position,
    int256 sizeDelta,
    uint256 executionPrice
) internal pure returns (int256) {
    // Calculate P&L for the portion being closed
    int256 priceDelta = int256(executionPrice) - int256(position.avgEntryPrice);
    return (priceDelta * (-sizeDelta)) / int256(PRECISION);
}
```

### Collateral Management
```solidity
struct CollateralConfig {
    bool isEnabled;
    uint256 weight;          // 100% = 10000
    address priceFeed;
    uint256 decimals;
}

function _getCollateralValue(
    address token,
    uint256 amount,
    uint256 weight
) internal view returns (uint256) {
    uint256 tokenPrice = oracle.getPrice(collateralConfigs[token].priceFeed);
    uint256 rawValue = (amount * tokenPrice) / (10 ** collateralConfigs[token].decimals);
    return (rawValue * weight) / 10000;
}
```

### Security Measures

1. **Position Validation**: Maximum leverage and size checks
2. **Margin Requirements**: Continuous monitoring
3. **Funding Rate Caps**: ±1% per hour maximum
4. **Arithmetic Safety**: Overflow/underflow protection

### Testing Requirements

1. **Unit Tests**:
   - Position lifecycle (open, increase, decrease, close)
   - Margin operations (add, remove, calculate)
   - Funding rate calculations and settlement
   - P&L calculations (realized and unrealized)

2. **Integration Tests**:
   - Multi-collateral scenarios
   - Funding rate impact on positions
   - Liquidation threshold calculations

3. **Edge Cases**:
   - Zero positions
   - Maximum leverage positions
   - Funding rate extremes

## Implementation Checklist

- [ ] PositionLib.sol - Position data structures
- [ ] MathLib.sol - Fixed-point math utilities
- [ ] PositionManager.sol - Position lifecycle
- [ ] MarginManager.sol - Margin operations
- [ ] FundingRateEngine.sol - Funding calculations
- [ ] PnLCalculator.sol - P&L logic
- [ ] CollateralManager.sol - Multi-collateral support
- [ ] Unit tests for each component
- [ ] Integration tests
- [ ] Gas optimization
- [ ] Security review

## Dependencies

- Oracle Module (for price feeds)
- Market Registry (for market parameters)
- Trading Engine (calls position updates)

## Estimated Implementation Time

- Core Implementation: 30-40 hours
- Testing: 15-20 hours
- Optimization: 10-15 hours
- Total: 55-75 hours