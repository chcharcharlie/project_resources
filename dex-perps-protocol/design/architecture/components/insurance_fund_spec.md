# Insurance Fund Module Specification

## Overview
The Insurance Fund Module manages the protocol's insurance fund, which covers bad debt from liquidations and maintains system solvency. It includes mechanisms for fund management, auto-deleveraging (ADL) when the fund is depleted, and revenue distribution.

## Component Structure

```
InsuranceFund/
├── contracts/
│   ├── InsuranceFund.sol        // Main fund management
│   ├── AutoDeleveraging.sol     // ADL mechanism
│   └── FundAllocator.sol        // Revenue distribution logic
├── interfaces/
│   ├── IInsuranceFund.sol       // Main interface
│   └── IAutoDeleveraging.sol    // ADL interface
└── libraries/
    ├── FundLib.sol              // Fund calculation utilities
    └── ADLLib.sol               // ADL ranking algorithms
```

## Technical Specifications

### Data Structures

```solidity
// Insurance fund configuration
struct InsuranceFundConfig {
    uint256 targetRatio;         // Target fund size as % of open interest (basis points)
    uint256 maxUtilization;      // Max % of fund that can be used at once
    address treasury;            // Excess funds destination
    bool adlEnabled;            // Whether ADL is enabled
}

// Fund allocation from different sources
struct FundAllocation {
    uint256 tradingFees;        // 40% of trading fees
    uint256 liquidationFees;    // 1% liquidation penalties
    uint256 directDeposits;     // Protocol seeding
    uint256 interestEarned;     // From fund investment (future)
}

// Market-specific fund tracking
struct MarketFund {
    uint256 balance;            // Fund balance for this market
    uint256 totalCovered;       // Total bad debt covered
    uint256 lastUpdateTime;     // Last balance update
    FundAllocation allocations; // Revenue breakdown
}

// Auto-deleveraging candidate
struct ADLCandidate {
    address trader;
    int256 pnl;                // Profit/loss
    uint256 leverage;          // Current leverage
    uint256 score;             // ADL priority score
    uint256 positionSize;      // Position size
}

// ADL event tracking
struct ADLEvent {
    address market;
    address deleveragedTrader;
    address counterparty;
    uint256 size;
    uint256 price;
    uint256 timestamp;
}
```

### Core Functions

#### Fund Management
```solidity
function addFunds(
    address market,
    uint256 amount,
    FundSource source
) external onlyAuthorized {
    MarketFund storage fund = marketFunds[market];
    
    // 1. Update fund balance
    fund.balance += amount;
    
    // 2. Track allocation source
    if (source == FundSource.TradingFees) {
        fund.allocations.tradingFees += amount;
    } else if (source == FundSource.LiquidationFees) {
        fund.allocations.liquidationFees += amount;
    } else if (source == FundSource.DirectDeposit) {
        fund.allocations.directDeposits += amount;
    }
    
    // 3. Update global fund balance
    totalFundBalance += amount;
    
    // 4. Check if fund exceeds target
    uint256 targetBalance = _calculateTargetBalance(market);
    if (fund.balance > targetBalance * 2) {
        // Send excess to treasury
        uint256 excess = fund.balance - targetBalance;
        fund.balance = targetBalance;
        totalFundBalance -= excess;
        IERC20(collateralToken).safeTransfer(config.treasury, excess);
        emit ExcessFundsTransferred(market, excess);
    }
    
    fund.lastUpdateTime = block.timestamp;
    emit FundsAdded(market, amount, source);
}

function coverBadDebt(
    address market,
    uint256 amount,
    address liquidatedTrader
) external onlyLiquidationEngine returns (bool covered) {
    MarketFund storage fund = marketFunds[market];
    
    // 1. Check fund balance
    if (fund.balance >= amount) {
        // Fund can cover the bad debt
        fund.balance -= amount;
        fund.totalCovered += amount;
        totalFundBalance -= amount;
        
        emit BadDebtCovered(market, liquidatedTrader, amount);
        return true;
    }
    
    // 2. Partial coverage if possible
    uint256 maxCoverage = (fund.balance * config.maxUtilization) / 10000;
    if (maxCoverage > 0) {
        fund.balance -= maxCoverage;
        fund.totalCovered += maxCoverage;
        totalFundBalance -= maxCoverage;
        
        uint256 uncovered = amount - maxCoverage;
        emit PartialBadDebtCovered(market, liquidatedTrader, maxCoverage, uncovered);
        
        // 3. Trigger ADL for remaining amount
        if (config.adlEnabled) {
            _triggerAutoDeleveraging(market, uncovered, liquidatedTrader);
        }
        
        return false;
    }
    
    // 4. No coverage available - full ADL
    emit BadDebtNotCovered(market, liquidatedTrader, amount);
    
    if (config.adlEnabled) {
        _triggerAutoDeleveraging(market, amount, liquidatedTrader);
    }
    
    return false;
}

function _calculateTargetBalance(address market) internal view returns (uint256) {
    (uint256 longOI, uint256 shortOI) = coreProtocol.getOpenInterest(market);
    uint256 totalOI = longOI + shortOI;
    return (totalOI * config.targetRatio) / 10000;
}
```

#### Auto-Deleveraging System
```solidity
function _triggerAutoDeleveraging(
    address market,
    uint256 amountNeeded,
    address liquidatedTrader
) internal {
    // 1. Get the liquidated position details
    Position memory liquidatedPos = coreProtocol.getPosition(liquidatedTrader, market);
    bool isLiquidatedLong = liquidatedPos.size > 0;
    
    // 2. Get profitable traders on opposite side
    ADLCandidate[] memory candidates = _getADLCandidates(
        market,
        !isLiquidatedLong, // opposite side
        amountNeeded
    );
    
    // 3. Execute ADL
    uint256 remainingAmount = amountNeeded;
    uint256 markPrice = oracle.getMarkPrice(market);
    
    for (uint i = 0; i < candidates.length && remainingAmount > 0; i++) {
        ADLCandidate memory candidate = candidates[i];
        
        // Calculate deleveraging size
        uint256 candidateNotional = (candidate.positionSize * markPrice) / PRECISION;
        uint256 deleverageSize;
        
        if (candidateNotional >= remainingAmount) {
            // Partial deleveraging of this position
            deleverageSize = (remainingAmount * PRECISION) / markPrice;
            remainingAmount = 0;
        } else {
            // Full deleveraging of this position
            deleverageSize = candidate.positionSize;
            remainingAmount -= candidateNotional;
        }
        
        // Execute deleveraging
        _executeADL(
            market,
            candidate.trader,
            deleverageSize,
            markPrice,
            liquidatedTrader
        );
    }
    
    require(remainingAmount == 0, "Insufficient ADL liquidity");
}

function _getADLCandidates(
    address market,
    bool getShortsOnly,
    uint256 amountNeeded
) internal view returns (ADLCandidate[] memory) {
    // 1. Get all traders with positions in the market
    address[] memory traders = coreProtocol.getMarketTraders(market);
    ADLCandidate[] memory allCandidates = new ADLCandidate[](traders.length);
    uint256 candidateCount = 0;
    
    // 2. Filter and score candidates
    for (uint i = 0; i < traders.length; i++) {
        Position memory pos = coreProtocol.getPosition(traders[i], market);
        
        // Skip if wrong side
        if (getShortsOnly && pos.size > 0) continue;
        if (!getShortsOnly && pos.size < 0) continue;
        if (pos.size == 0) continue;
        
        // Calculate PnL and leverage
        int256 pnl = coreProtocol.getUnrealizedPnL(pos, oracle.getMarkPrice(market));
        uint256 leverage = coreProtocol.getPositionLeverage(traders[i], market);
        
        // Only include profitable positions
        if (pnl > 0) {
            allCandidates[candidateCount] = ADLCandidate({
                trader: traders[i],
                pnl: pnl,
                leverage: leverage,
                score: _calculateADLScore(pnl, leverage),
                positionSize: _abs(pos.size)
            });
            candidateCount++;
        }
    }
    
    // 3. Sort by ADL score (highest first)
    ADLCandidate[] memory candidates = new ADLCandidate[](candidateCount);
    for (uint i = 0; i < candidateCount; i++) {
        candidates[i] = allCandidates[i];
    }
    _sortByScore(candidates);
    
    // 4. Return enough candidates to cover amount needed
    uint256 accumulated = 0;
    uint256 resultCount = 0;
    uint256 markPrice = oracle.getMarkPrice(market);
    
    for (uint i = 0; i < candidates.length && accumulated < amountNeeded; i++) {
        accumulated += (candidates[i].positionSize * markPrice) / PRECISION;
        resultCount++;
    }
    
    ADLCandidate[] memory result = new ADLCandidate[](resultCount);
    for (uint i = 0; i < resultCount; i++) {
        result[i] = candidates[i];
    }
    
    return result;
}

function _calculateADLScore(int256 pnl, uint256 leverage) internal pure returns (uint256) {
    // ADL Score = PnL Ranking × Leverage Ranking
    // Higher score = higher priority for ADL
    
    // Normalize PnL to 0-100 scale
    uint256 pnlScore = uint256(pnl) / 1e18; // Simplified, should use proper scaling
    
    // Leverage score (higher leverage = higher score)
    uint256 leverageScore = leverage;
    
    return (pnlScore * leverageScore) / 100;
}

function _executeADL(
    address market,
    address trader,
    uint256 size,
    uint256 price,
    address counterparty
) internal {
    // 1. Force close position at current mark price
    coreProtocol.forceClosePosition(trader, market, size, price);
    
    // 2. Record ADL event
    adlEvents.push(ADLEvent({
        market: market,
        deleveragedTrader: trader,
        counterparty: counterparty,
        size: size,
        price: price,
        timestamp: block.timestamp
    }));
    
    // 3. Notify affected trader
    emit PositionAutoDeleveraged(
        market,
        trader,
        size,
        price,
        counterparty
    );
}
```

#### Fund Analytics
```solidity
function getFundMetrics(address market) external view returns (
    uint256 balance,
    uint256 targetBalance,
    uint256 utilizationRate,
    uint256 coverageRatio,
    uint256 totalBadDebtCovered
) {
    MarketFund storage fund = marketFunds[market];
    balance = fund.balance;
    targetBalance = _calculateTargetBalance(market);
    
    // Utilization rate: how much of the fund has been used
    if (fund.totalCovered > 0) {
        utilizationRate = (fund.totalCovered * 10000) / 
            (fund.totalCovered + fund.balance);
    }
    
    // Coverage ratio: fund size vs open interest
    (uint256 longOI, uint256 shortOI) = coreProtocol.getOpenInterest(market);
    uint256 totalOI = longOI + shortOI;
    if (totalOI > 0) {
        coverageRatio = (fund.balance * 10000) / totalOI;
    }
    
    totalBadDebtCovered = fund.totalCovered;
}

function getGlobalFundMetrics() external view returns (
    uint256 totalBalance,
    uint256 totalCovered,
    uint256 adlEventsCount,
    uint256 averageCoverageRatio
) {
    totalBalance = totalFundBalance;
    
    address[] memory markets = marketRegistry.getAllMarkets();
    uint256 totalOI;
    
    for (uint i = 0; i < markets.length; i++) {
        totalCovered += marketFunds[markets[i]].totalCovered;
        
        (uint256 longOI, uint256 shortOI) = coreProtocol.getOpenInterest(markets[i]);
        totalOI += longOI + shortOI;
    }
    
    adlEventsCount = adlEvents.length;
    
    if (totalOI > 0) {
        averageCoverageRatio = (totalBalance * 10000) / totalOI;
    }
}
```

#### Revenue Distribution
```solidity
contract FundAllocator {
    struct AllocationConfig {
        uint256 insuranceFundShare;  // 40% default
        uint256 treasuryShare;       // 40% default
        uint256 stakingRewards;      // 20% default
    }
    
    function distributeFees(
        address market,
        uint256 tradingFees,
        FeeType feeType
    ) external onlyTradingEngine {
        AllocationConfig memory config = allocationConfigs[market];
        
        // 1. Insurance fund allocation
        uint256 insuranceAmount = (tradingFees * config.insuranceFundShare) / 10000;
        if (insuranceAmount > 0) {
            IERC20(collateralToken).safeTransfer(insuranceFund, insuranceAmount);
            IInsuranceFund(insuranceFund).addFunds(
                market,
                insuranceAmount,
                FundSource.TradingFees
            );
        }
        
        // 2. Treasury allocation
        uint256 treasuryAmount = (tradingFees * config.treasuryShare) / 10000;
        if (treasuryAmount > 0) {
            IERC20(collateralToken).safeTransfer(treasury, treasuryAmount);
        }
        
        // 3. Staking rewards (future implementation)
        uint256 stakingAmount = tradingFees - insuranceAmount - treasuryAmount;
        if (stakingAmount > 0) {
            IERC20(collateralToken).safeTransfer(stakingRewards, stakingAmount);
        }
        
        emit FeesDistributed(market, tradingFees, insuranceAmount, treasuryAmount, stakingAmount);
    }
}
```

### Security Measures

1. **Access Control**: Only authorized contracts can use funds
2. **Maximum Utilization**: Prevents draining entire fund at once
3. **ADL Safeguards**: Only affects profitable traders
4. **Excess Fund Management**: Automatic treasury transfers

### Testing Requirements

1. **Unit Tests**:
   - Fund addition and tracking
   - Bad debt coverage scenarios
   - ADL candidate selection
   - Revenue distribution

2. **Integration Tests**:
   - Liquidation to fund coverage flow
   - ADL execution with position closure
   - Multi-market fund management

3. **Stress Tests**:
   - Mass liquidation scenarios
   - Fund depletion and recovery
   - ADL cascade effects

## Implementation Checklist

- [ ] FundLib.sol - Fund calculation utilities
- [ ] ADLLib.sol - ADL algorithms
- [ ] InsuranceFund.sol - Main fund logic
- [ ] AutoDeleveraging.sol - ADL implementation
- [ ] FundAllocator.sol - Fee distribution
- [ ] Unit tests for each component
- [ ] Integration tests
- [ ] Gas optimization
- [ ] Security review

## Dependencies

- Core Protocol Engine (position data)
- Liquidation Engine (bad debt amounts)
- Trading Engine (fee collection)
- Oracle Module (prices for ADL)

## Estimated Implementation Time

- Core Implementation: 12-15 hours
- Testing: 6-8 hours
- Optimization: 3-5 hours
- Total: 21-28 hours