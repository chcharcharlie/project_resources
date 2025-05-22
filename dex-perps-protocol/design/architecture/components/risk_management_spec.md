# Risk Management Module Specification

## Overview
The Risk Management Module monitors system-wide risk and executes protective actions including liquidations, circuit breakers, and position limits. This module ensures system solvency and protects against extreme market conditions.

## Component Structure

```
RiskManagement/
├── contracts/
│   ├── LiquidationEngine.sol    // Liquidation execution logic
│   ├── CircuitBreaker.sol       // Emergency market controls
│   ├── PositionLimits.sol       // Risk limit enforcement
│   └── RiskCalculator.sol       // Risk metric calculations
├── interfaces/
│   ├── IRiskManagement.sol      // Main interface
│   └── ILiquidationEngine.sol   // Liquidation interface
└── libraries/
    ├── LiquidationLib.sol       // Liquidation utilities
    └── RiskMetricsLib.sol       // Risk calculations
```

## Technical Specifications

### Data Structures

```solidity
// Liquidation configuration
struct LiquidationConfig {
    uint256 partialLiquidationThreshold;  // 50,000 USD notional
    uint256 partialLiquidationRatio;      // 50% of position
    uint256 liquidationPenalty;           // 15 (1.5% in basis points)
    uint256 liquidatorReward;             // 5 (0.5% in basis points)
    uint256 insuranceFundShare;           // 10 (1.0% in basis points)
    uint256 priorityPeriod;               // 100ms for whitelisted liquidators
}

// Circuit breaker configuration
struct CircuitBreakerConfig {
    uint256 priceDeviationShort;     // 5% in 5 minutes
    uint256 priceDeviationMedium;    // 10% in 1 hour
    uint256 priceDeviationLong;      // 20% in 24 hours
    uint256 cooldownPeriod;          // 5 minutes
    uint256 volumeSpikeThreshold;    // 10x average volume
}

// Market risk state
struct MarketRiskState {
    CircuitBreakerStatus status;     // Normal, Warning, Halted, Cooldown
    uint256 haltedAt;                // Timestamp when halted
    uint256 lastPriceCheckpoint;     // For deviation calculation
    uint256[3] priceHistory;         // [5min, 1hr, 24hr] prices
    uint256 averageVolume;           // Rolling average volume
}

// Position risk metrics
struct PositionRisk {
    uint256 notionalValue;
    uint256 marginRatio;
    uint256 liquidationPrice;
    uint256 maintenanceMargin;
    uint256 initialMargin;
    bool isLiquidatable;
}

// Liquidation event tracking
struct LiquidationEvent {
    address trader;
    address liquidator;
    uint256 size;
    uint256 price;
    uint256 penalty;
    uint256 timestamp;
    bool isPartial;
}
```

### Core Functions

#### Liquidation Engine
```solidity
function liquidatePosition(
    address trader,
    address market
) external returns (uint256 liquidatedSize, uint256 penalty) {
    // 1. Verify position is liquidatable
    Position memory position = coreProtocol.getPosition(trader, market);
    PositionRisk memory risk = _calculatePositionRisk(position, market);
    require(risk.isLiquidatable, "Position not liquidatable");
    
    // 2. Check liquidator eligibility
    bool isPriorityLiquidator = hasRole(LIQUIDATOR_ROLE, msg.sender);
    if (isPriorityLiquidator) {
        // Priority liquidators get first 100ms
        require(
            liquidationTimestamps[trader][market] == 0 ||
            block.timestamp >= liquidationTimestamps[trader][market] + config.priorityPeriod,
            "Priority period active"
        );
    } else {
        // Public liquidators must wait
        require(
            liquidationTimestamps[trader][market] > 0 &&
            block.timestamp >= liquidationTimestamps[trader][market] + config.priorityPeriod,
            "Not eligible yet"
        );
    }
    
    // 3. Determine liquidation type and size
    uint256 positionNotional = risk.notionalValue;
    bool isPartial = positionNotional > config.partialLiquidationThreshold;
    
    if (isPartial) {
        // Partial liquidation
        liquidatedSize = (position.size * config.partialLiquidationRatio) / 10000;
    } else {
        // Full liquidation
        liquidatedSize = position.size;
    }
    
    // 4. Calculate penalties
    uint256 markPrice = oracle.getMarkPrice(market);
    uint256 liquidationValue = (liquidatedSize * markPrice) / PRECISION;
    penalty = (liquidationValue * config.liquidationPenalty) / 10000;
    
    uint256 liquidatorReward = (liquidationValue * config.liquidatorReward) / 10000;
    uint256 insuranceFundContribution = penalty - liquidatorReward;
    
    // 5. Execute liquidation through trading engine
    uint256 orderId = tradingEngine.placeLiquidationOrder(
        market,
        trader,
        liquidatedSize,
        markPrice
    );
    
    // 6. Update position and distribute rewards
    coreProtocol.reducePo

(trader, market, liquidatedSize, penalty);
    _payLiquidator(msg.sender, liquidatorReward);
    insuranceFund.addFunds(insuranceFundContribution);
    
    // 7. Track liquidation event
    _recordLiquidation(LiquidationEvent({
        trader: trader,
        liquidator: msg.sender,
        size: liquidatedSize,
        price: markPrice,
        penalty: penalty,
        timestamp: block.timestamp,
        isPartial: isPartial
    }));
    
    // 8. Check if position needs further liquidation
    if (isPartial) {
        Position memory remainingPosition = coreProtocol.getPosition(trader, market);
        PositionRisk memory remainingRisk = _calculatePositionRisk(remainingPosition, market);
        if (remainingRisk.isLiquidatable) {
            // Mark for continued liquidation
            liquidationTimestamps[trader][market] = block.timestamp;
        }
    }
    
    emit PositionLiquidated(
        trader,
        market,
        msg.sender,
        liquidatedSize,
        markPrice,
        penalty,
        isPartial
    );
}

function _calculatePositionRisk(
    Position memory position,
    address market
) internal view returns (PositionRisk memory risk) {
    if (position.size == 0) {
        return risk;
    }
    
    MarketConfig memory config = marketRegistry.getMarketConfig(market);
    uint256 markPrice = oracle.getMarkPrice(market);
    
    // Calculate notional value
    risk.notionalValue = (_abs(position.size) * markPrice) / PRECISION;
    
    // Calculate maintenance margin requirement
    risk.maintenanceMargin = (risk.notionalValue * config.maintenanceMarginRate) / 10000;
    risk.initialMargin = (risk.notionalValue * config.initialMarginRate) / 10000;
    
    // Calculate effective margin (including unrealized P&L)
    int256 unrealizedPnL = coreProtocol.getUnrealizedPnL(position, markPrice);
    uint256 effectiveMargin = uint256(int256(position.margin) + unrealizedPnL);
    
    // Calculate margin ratio
    risk.marginRatio = (effectiveMargin * 10000) / risk.notionalValue;
    
    // Check if liquidatable
    risk.isLiquidatable = effectiveMargin < risk.maintenanceMargin;
    
    // Calculate liquidation price
    if (position.size > 0) {
        // Long position
        risk.liquidationPrice = position.avgEntryPrice * 
            (10000 - config.initialMarginRate + config.maintenanceMarginRate) / 10000;
    } else {
        // Short position
        risk.liquidationPrice = position.avgEntryPrice * 
            (10000 + config.initialMarginRate - config.maintenanceMarginRate) / 10000;
    }
}
```

#### Circuit Breaker System
```solidity
function checkAndUpdateCircuitBreaker(address market) external {
    MarketRiskState storage state = marketRiskStates[market];
    CircuitBreakerConfig memory config = circuitBreakerConfigs[market];
    
    // 1. Get current price
    uint256 currentPrice = oracle.getMarkPrice(market);
    uint256 currentTime = block.timestamp;
    
    // 2. Update price history
    if (currentTime >= state.lastPriceCheckpoint + 5 minutes) {
        // Shift price history
        state.priceHistory[2] = state.priceHistory[1]; // 24hr -> old
        state.priceHistory[1] = state.priceHistory[0]; // 1hr -> 24hr
        state.priceHistory[0] = state.lastPriceCheckpoint; // 5min -> 1hr
        state.lastPriceCheckpoint = currentPrice;
    }
    
    // 3. Check if in cooldown
    if (state.status == CircuitBreakerStatus.Cooldown) {
        if (currentTime >= state.haltedAt + config.cooldownPeriod) {
            state.status = CircuitBreakerStatus.Normal;
            emit CircuitBreakerReset(market);
        }
        return;
    }
    
    // 4. Calculate price deviations
    uint256 deviation5m = _calculateDeviation(currentPrice, state.priceHistory[0]);
    uint256 deviation1h = _calculateDeviation(currentPrice, state.priceHistory[1]);
    uint256 deviation24h = _calculateDeviation(currentPrice, state.priceHistory[2]);
    
    // 5. Check deviation thresholds
    bool shouldHalt = false;
    string memory reason;
    
    if (deviation5m > config.priceDeviationShort) {
        shouldHalt = true;
        reason = "5min deviation exceeded";
    } else if (deviation1h > config.priceDeviationMedium) {
        shouldHalt = true;
        reason = "1hr deviation exceeded";
    } else if (deviation24h > config.priceDeviationLong) {
        shouldHalt = true;
        reason = "24hr deviation exceeded";
    }
    
    // 6. Check volume spike
    uint256 currentVolume = tradingEngine.getVolume(market, 5 minutes);
    if (currentVolume > state.averageVolume * config.volumeSpikeThreshold / 10000) {
        shouldHalt = true;
        reason = "Volume spike detected";
    }
    
    // 7. Execute circuit breaker if needed
    if (shouldHalt && state.status == CircuitBreakerStatus.Normal) {
        state.status = CircuitBreakerStatus.Halted;
        state.haltedAt = currentTime;
        
        // Execute protective actions
        _haltMarket(market);
        
        emit CircuitBreakerTriggered(market, reason, currentPrice);
    }
    
    // 8. Update rolling averages
    state.averageVolume = (state.averageVolume * 95 + currentVolume * 5) / 100;
}

function _haltMarket(address market) internal {
    // 1. Cancel all open orders
    tradingEngine.cancelAllMarketOrders(market);
    
    // 2. Prevent new position opening
    marketRegistry.setMarketStatus(market, MarketStatus.PositionCloseOnly);
    
    // 3. Reduce leverage limits
    marketRegistry.setMaxLeverage(market, 10); // Reduce to 10x
    
    // 4. Widen spread requirements
    tradingEngine.setMinimumSpread(market, 100); // 1% minimum spread
}
```

#### Position Limits
```solidity
struct PositionLimitConfig {
    uint256 maxPositionSize;      // Per trader
    uint256 maxMarketOI;          // Total open interest
    uint256 maxConcentration;     // Max % of OI per trader
    uint256 maxOrderSize;         // Single order limit
}

function checkPositionLimits(
    address trader,
    address market,
    int256 sizeDelta
) external view returns (bool allowed, string memory reason) {
    PositionLimitConfig memory limits = positionLimits[market];
    Position memory currentPosition = coreProtocol.getPosition(trader, market);
    
    // 1. Check max position size
    uint256 newPositionSize = _abs(currentPosition.size + sizeDelta);
    if (newPositionSize > limits.maxPositionSize) {
        return (false, "Exceeds max position size");
    }
    
    // 2. Check order size
    if (_abs(sizeDelta) > limits.maxOrderSize) {
        return (false, "Order too large");
    }
    
    // 3. Check market open interest
    (uint256 longOI, uint256 shortOI) = coreProtocol.getOpenInterest(market);
    uint256 totalOI = longOI + shortOI;
    
    uint256 oiDelta = _abs(sizeDelta);
    if (totalOI + oiDelta > limits.maxMarketOI) {
        return (false, "Exceeds market OI limit");
    }
    
    // 4. Check concentration
    uint256 traderOI = _abs(currentPosition.size + sizeDelta);
    uint256 concentration = (traderOI * 10000) / (totalOI + oiDelta);
    if (concentration > limits.maxConcentration) {
        return (false, "Exceeds concentration limit");
    }
    
    return (true, "");
}

function updatePositionLimits(
    address market,
    PositionLimitConfig memory newLimits
) external onlyAdmin {
    require(newLimits.maxPositionSize > 0, "Invalid position size");
    require(newLimits.maxMarketOI > 0, "Invalid market OI");
    require(newLimits.maxConcentration <= 2000, "Concentration too high"); // Max 20%
    
    positionLimits[market] = newLimits;
    emit PositionLimitsUpdated(market, newLimits);
}
```

### Risk Monitoring Functions
```solidity
function getSystemRiskMetrics() external view returns (
    uint256 totalOpenInterest,
    uint256 totalCollateral,
    uint256 insuranceFundBalance,
    uint256 liquidationsLast24h,
    uint256 utilizationRate
) {
    // Aggregate metrics across all markets
    address[] memory markets = marketRegistry.getAllMarkets();
    
    for (uint i = 0; i < markets.length; i++) {
        (uint256 longOI, uint256 shortOI) = coreProtocol.getOpenInterest(markets[i]);
        totalOpenInterest += longOI + shortOI;
    }
    
    totalCollateral = coreProtocol.getTotalCollateral();
    insuranceFundBalance = insuranceFund.getBalance();
    liquidationsLast24h = _getLiquidationCount(24 hours);
    utilizationRate = (totalOpenInterest * 10000) / totalCollateral;
}

function getMarketRiskMetrics(address market) external view returns (
    uint256 longOpenInterest,
    uint256 shortOpenInterest,
    int256 fundingRate,
    uint256 liquidationThreshold,
    CircuitBreakerStatus cbStatus
) {
    (longOpenInterest, shortOpenInterest) = coreProtocol.getOpenInterest(market);
    fundingRate = coreProtocol.getCurrentFundingRate(market);
    
    // Calculate average liquidation threshold
    uint256 totalPositions = coreProtocol.getActivePositionCount(market);
    uint256 liquidatablePositions = _countLiquidatablePositions(market);
    liquidationThreshold = (liquidatablePositions * 10000) / totalPositions;
    
    cbStatus = marketRiskStates[market].status;
}
```

### Security Measures

1. **Access Control**: Role-based permissions for liquidators
2. **Reentrancy Protection**: Guards on all state-changing functions  
3. **Oracle Validation**: Sanity checks on price feeds
4. **Parameter Bounds**: Limits on all configurable parameters

### Testing Requirements

1. **Unit Tests**:
   - Liquidation calculations and execution
   - Circuit breaker trigger conditions
   - Position limit enforcement
   - Risk metric calculations

2. **Integration Tests**:
   - Liquidation with trading engine
   - Circuit breaker market impact
   - Multi-market risk scenarios

3. **Stress Tests**:
   - Mass liquidation scenarios
   - Rapid price movement handling
   - High volume circuit breakers

## Implementation Checklist

- [ ] LiquidationLib.sol - Liquidation utilities
- [ ] RiskMetricsLib.sol - Risk calculations
- [ ] LiquidationEngine.sol - Liquidation logic
- [ ] CircuitBreaker.sol - Market halting
- [ ] PositionLimits.sol - Limit enforcement
- [ ] RiskCalculator.sol - Risk metrics
- [ ] Unit tests for each component
- [ ] Integration tests
- [ ] Gas optimization
- [ ] Security review

## Dependencies

- Core Protocol Engine (position data)
- Trading Engine (order cancellation)
- Oracle Module (price feeds)
- Insurance Fund (penalty distribution)
- Market Registry (parameter updates)

## Estimated Implementation Time

- Core Implementation: 20-25 hours
- Testing: 10-15 hours
- Optimization: 5-10 hours
- Total: 35-50 hours