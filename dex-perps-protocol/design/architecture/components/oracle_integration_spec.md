# Oracle Integration Module Specification

## Overview
The Oracle Integration Module aggregates price data from multiple oracle sources, validates the data, and provides reliable price feeds to the protocol. This module is critical for accurate mark prices, liquidations, and funding rate calculations.

## Component Structure

```
OracleIntegration/
├── contracts/
│   ├── OracleAggregator.sol     // Main aggregation logic
│   ├── OracleValidator.sol      // Price validation
│   ├── ChainlinkAdapter.sol     // Chainlink integration
│   ├── PythAdapter.sol          // Pyth Network integration
│   └── FallbackOracle.sol       // Emergency price source
├── interfaces/
│   ├── IOracleAggregator.sol    // Main interface
│   └── IPriceOracle.sol         // Oracle adapter interface
└── libraries/
    ├── PriceLib.sol             // Price manipulation utilities
    └── StatisticsLib.sol        // Statistical calculations
```

## Technical Specifications

### Data Structures

```solidity
// Oracle configuration
struct OracleConfig {
    address oracleAddress;
    uint256 weight;              // Weight in aggregation (basis points)
    uint256 maxDeviation;        // Max acceptable deviation from median
    uint256 maxStaleness;        // Maximum age of price data
    bool isEnabled;
    OracleType oracleType;       // CHAINLINK, PYTH, CUSTOM
}

// Price data point
struct PriceData {
    uint256 price;
    uint256 confidence;          // Confidence interval/deviation
    uint256 timestamp;
    address source;
    bool isValid;
}

// Aggregated price result
struct AggregatedPrice {
    uint256 price;               // Final aggregated price
    uint256 confidence;          // System confidence in price
    uint256 timestamp;
    uint256 numSources;          // Number of valid sources
}

// Oracle performance metrics
struct OracleMetrics {
    uint256 totalUpdates;
    uint256 failedUpdates;
    uint256 averageDeviation;    // From aggregated price
    uint256 lastUpdateTime;
    uint256 uptime;              // Percentage in basis points
}

// Market oracle configuration
struct MarketOracleConfig {
    bytes32 priceId;             // Price identifier (e.g., "AAPL/USD")
    OracleConfig[] oracles;      // Array of oracle configurations
    uint256 minOracles;          // Minimum valid oracles required
    uint256 deviationThreshold;  // Max deviation for outlier detection
}
```

### Core Functions

#### Price Aggregation
```solidity
function getPrice(address market) external view returns (
    uint256 price,
    uint256 confidence,
    uint256 timestamp
) {
    MarketOracleConfig storage config = marketOracleConfigs[market];
    require(config.minOracles > 0, "Market not configured");
    
    // 1. Collect prices from all sources
    PriceData[] memory prices = _collectPrices(config);
    
    // 2. Validate price data
    uint256 validCount = _validatePrices(prices, config);
    require(validCount >= config.minOracles, "Insufficient valid prices");
    
    // 3. Filter outliers
    PriceData[] memory filtered = _filterOutliers(prices, config.deviationThreshold);
    require(filtered.length >= config.minOracles, "Too many outliers");
    
    // 4. Aggregate prices
    AggregatedPrice memory aggregated = _aggregatePrices(filtered, config);
    
    // 5. Final validation
    require(aggregated.price > 0, "Invalid aggregated price");
    require(block.timestamp - aggregated.timestamp <= MAX_PRICE_AGE, "Price too stale");
    
    return (aggregated.price, aggregated.confidence, aggregated.timestamp);
}

function _collectPrices(
    MarketOracleConfig storage config
) internal view returns (PriceData[] memory prices) {
    prices = new PriceData[](config.oracles.length);
    
    for (uint i = 0; i < config.oracles.length; i++) {
        OracleConfig memory oracle = config.oracles[i];
        
        if (!oracle.isEnabled) {
            continue;
        }
        
        try IPriceOracle(oracle.oracleAddress).getPrice(config.priceId) 
        returns (uint256 price, uint256 confidence, uint256 timestamp) {
            prices[i] = PriceData({
                price: price,
                confidence: confidence,
                timestamp: timestamp,
                source: oracle.oracleAddress,
                isValid: true
            });
        } catch {
            // Oracle call failed, mark as invalid
            prices[i].isValid = false;
            _recordOracleFailure(oracle.oracleAddress);
        }
    }
}

function _filterOutliers(
    PriceData[] memory prices,
    uint256 deviationThreshold
) internal pure returns (PriceData[] memory filtered) {
    // 1. Calculate median price
    uint256 median = _calculateMedian(prices);
    
    // 2. Calculate Median Absolute Deviation (MAD)
    uint256 mad = _calculateMAD(prices, median);
    
    // 3. Filter outliers using MAD method
    uint256 count = 0;
    PriceData[] memory temp = new PriceData[](prices.length);
    
    for (uint i = 0; i < prices.length; i++) {
        if (!prices[i].isValid) continue;
        
        uint256 deviation = _abs(prices[i].price, median);
        uint256 normalizedDeviation = (deviation * 10000) / mad;
        
        if (normalizedDeviation <= deviationThreshold) {
            temp[count] = prices[i];
            count++;
        }
    }
    
    // 4. Copy to filtered array
    filtered = new PriceData[](count);
    for (uint i = 0; i < count; i++) {
        filtered[i] = temp[i];
    }
}

function _aggregatePrices(
    PriceData[] memory prices,
    MarketOracleConfig storage config
) internal view returns (AggregatedPrice memory result) {
    uint256 weightedSum = 0;
    uint256 totalWeight = 0;
    uint256 minConfidence = type(uint256).max;
    uint256 latestTimestamp = 0;
    
    // Calculate weighted average
    for (uint i = 0; i < prices.length; i++) {
        if (!prices[i].isValid) continue;
        
        // Find oracle weight
        uint256 weight = 0;
        for (uint j = 0; j < config.oracles.length; j++) {
            if (config.oracles[j].oracleAddress == prices[i].source) {
                weight = config.oracles[j].weight;
                break;
            }
        }
        
        // Apply time decay to weight (more recent = higher weight)
        uint256 age = block.timestamp - prices[i].timestamp;
        uint256 timeWeight = weight * (3600 - Math.min(age, 3600)) / 3600;
        
        weightedSum += prices[i].price * timeWeight;
        totalWeight += timeWeight;
        
        // Track minimum confidence
        if (prices[i].confidence < minConfidence) {
            minConfidence = prices[i].confidence;
        }
        
        // Track latest timestamp
        if (prices[i].timestamp > latestTimestamp) {
            latestTimestamp = prices[i].timestamp;
        }
    }
    
    require(totalWeight > 0, "No valid weights");
    
    // Calculate agreement factor (how close prices are to each other)
    uint256 agreementFactor = _calculateAgreementFactor(prices);
    
    result = AggregatedPrice({
        price: weightedSum / totalWeight,
        confidence: (minConfidence * agreementFactor) / 10000,
        timestamp: latestTimestamp,
        numSources: prices.length
    });
}
```

#### Oracle Adapters
```solidity
// Chainlink Adapter
contract ChainlinkAdapter is IPriceOracle {
    mapping(bytes32 => address) public priceFeeds;
    
    function getPrice(bytes32 priceId) external view returns (
        uint256 price,
        uint256 confidence,
        uint256 timestamp
    ) {
        address feed = priceFeeds[priceId];
        require(feed != address(0), "Price feed not configured");
        
        (
            uint80 roundId,
            int256 answer,
            uint256 startedAt,
            uint256 updatedAt,
            uint80 answeredInRound
        ) = AggregatorV3Interface(feed).latestRoundData();
        
        require(answer > 0, "Invalid price");
        require(updatedAt > 0, "Round not complete");
        
        // Chainlink provides 8 decimal prices, scale to 18
        price = uint256(answer) * 10**10;
        
        // Estimate confidence based on price feed parameters
        confidence = 9900; // 99% confidence for Chainlink
        timestamp = updatedAt;
    }
}

// Pyth Network Adapter
contract PythAdapter is IPriceOracle {
    IPyth public pythContract;
    
    function getPrice(bytes32 priceId) external view returns (
        uint256 price,
        uint256 confidence,
        uint256 timestamp
    ) {
        PythStructs.Price memory pythPrice = pythContract.getPriceUnsafe(priceId);
        
        require(pythPrice.price > 0, "Invalid price");
        
        // Convert from Pyth format to standard format
        uint256 scaledPrice = _scalePrice(
            uint256(pythPrice.price),
            pythPrice.expo
        );
        
        // Calculate confidence from Pyth's confidence interval
        uint256 scaledConfidence = _scalePrice(
            uint256(pythPrice.conf),
            pythPrice.expo
        );
        uint256 confidenceRatio = 10000 - (scaledConfidence * 10000 / scaledPrice);
        
        return (scaledPrice, confidenceRatio, pythPrice.publishTime);
    }
    
    function _scalePrice(uint256 price, int32 expo) internal pure returns (uint256) {
        if (expo >= 0) {
            return price * (10 ** uint32(expo));
        } else {
            return price * 10**18 / (10 ** uint32(-expo));
        }
    }
}
```

#### Oracle Management
```solidity
function addOracle(
    address market,
    address oracle,
    uint256 weight,
    OracleType oracleType
) external onlyAdmin {
    MarketOracleConfig storage config = marketOracleConfigs[market];
    
    // Validate oracle
    require(oracle != address(0), "Invalid oracle address");
    require(weight > 0 && weight <= 10000, "Invalid weight");
    
    // Check if oracle already exists
    for (uint i = 0; i < config.oracles.length; i++) {
        require(config.oracles[i].oracleAddress != oracle, "Oracle already added");
    }
    
    // Add oracle configuration
    config.oracles.push(OracleConfig({
        oracleAddress: oracle,
        weight: weight,
        maxDeviation: 500, // 5% default
        maxStaleness: 3600, // 1 hour default
        isEnabled: true,
        oracleType: oracleType
    }));
    
    emit OracleAdded(market, oracle, weight);
}

function updateOracleWeight(
    address market,
    address oracle,
    uint256 newWeight
) external onlyOperator {
    MarketOracleConfig storage config = marketOracleConfigs[market];
    
    bool found = false;
    for (uint i = 0; i < config.oracles.length; i++) {
        if (config.oracles[i].oracleAddress == oracle) {
            config.oracles[i].weight = newWeight;
            found = true;
            break;
        }
    }
    
    require(found, "Oracle not found");
    emit OracleWeightUpdated(market, oracle, newWeight);
}

function disableOracle(
    address market,
    address oracle
) external onlyOperator {
    MarketOracleConfig storage config = marketOracleConfigs[market];
    
    uint256 enabledCount = 0;
    uint256 targetIndex = type(uint256).max;
    
    // Count enabled oracles and find target
    for (uint i = 0; i < config.oracles.length; i++) {
        if (config.oracles[i].isEnabled) {
            enabledCount++;
        }
        if (config.oracles[i].oracleAddress == oracle) {
            targetIndex = i;
        }
    }
    
    require(targetIndex != type(uint256).max, "Oracle not found");
    require(enabledCount > config.minOracles, "Cannot disable: minimum oracles required");
    
    config.oracles[targetIndex].isEnabled = false;
    emit OracleDisabled(market, oracle);
}
```

#### Fallback Mechanism
```solidity
contract FallbackOracle is IPriceOracle {
    mapping(bytes32 => uint256) public fallbackPrices;
    mapping(bytes32 => uint256) public lastUpdate;
    uint256 constant FALLBACK_VALIDITY = 15 minutes;
    
    function updateFallbackPrice(
        bytes32 priceId,
        uint256 price
    ) external onlyOperator {
        fallbackPrices[priceId] = price;
        lastUpdate[priceId] = block.timestamp;
        emit FallbackPriceUpdated(priceId, price);
    }
    
    function getPrice(bytes32 priceId) external view returns (
        uint256 price,
        uint256 confidence,
        uint256 timestamp
    ) {
        require(fallbackPrices[priceId] > 0, "No fallback price");
        require(
            block.timestamp - lastUpdate[priceId] <= FALLBACK_VALIDITY,
            "Fallback price expired"
        );
        
        return (
            fallbackPrices[priceId],
            5000, // 50% confidence for fallback
            lastUpdate[priceId]
        );
    }
}
```

### Statistical Functions
```solidity
library StatisticsLib {
    function calculateMedian(PriceData[] memory prices) 
        internal 
        pure 
        returns (uint256) 
    {
        uint256[] memory validPrices = _extractValidPrices(prices);
        _quickSort(validPrices, 0, validPrices.length - 1);
        
        uint256 len = validPrices.length;
        if (len % 2 == 0) {
            return (validPrices[len/2 - 1] + validPrices[len/2]) / 2;
        } else {
            return validPrices[len/2];
        }
    }
    
    function calculateMAD(
        PriceData[] memory prices,
        uint256 median
    ) internal pure returns (uint256) {
        uint256[] memory deviations = new uint256[](prices.length);
        uint256 count = 0;
        
        for (uint i = 0; i < prices.length; i++) {
            if (prices[i].isValid) {
                deviations[count] = _abs(prices[i].price, median);
                count++;
            }
        }
        
        // Get median of deviations
        _quickSort(deviations, 0, count - 1);
        if (count % 2 == 0) {
            return (deviations[count/2 - 1] + deviations[count/2]) / 2;
        } else {
            return deviations[count/2];
        }
    }
    
    function calculateAgreementFactor(PriceData[] memory prices) 
        internal 
        pure 
        returns (uint256) 
    {
        if (prices.length < 2) return 10000; // 100% if only one price
        
        uint256 sum = 0;
        uint256 count = 0;
        
        // Calculate mean
        for (uint i = 0; i < prices.length; i++) {
            if (prices[i].isValid) {
                sum += prices[i].price;
                count++;
            }
        }
        uint256 mean = sum / count;
        
        // Calculate standard deviation
        uint256 variance = 0;
        for (uint i = 0; i < prices.length; i++) {
            if (prices[i].isValid) {
                uint256 diff = _abs(prices[i].price, mean);
                variance += (diff * diff);
            }
        }
        
        uint256 stdDev = _sqrt(variance / count);
        uint256 coefficientOfVariation = (stdDev * 10000) / mean;
        
        // Agreement factor: 100% - coefficient of variation
        return 10000 > coefficientOfVariation ? 
            10000 - coefficientOfVariation : 0;
    }
}
```

### Security Measures

1. **Oracle Validation**: Multiple checks for price validity
2. **Outlier Detection**: Statistical methods to filter bad data
3. **Minimum Oracle Requirement**: Configurable minimum sources
4. **Staleness Protection**: Maximum age for price data
5. **Admin Controls**: Timelocked oracle configuration changes

### Testing Requirements

1. **Unit Tests**:
   - Price aggregation with various inputs
   - Outlier detection algorithms
   - Oracle adapter integrations
   - Fallback mechanism

2. **Integration Tests**:
   - Multi-oracle scenarios
   - Oracle failure handling
   - Price deviation scenarios

3. **Edge Cases**:
   - All oracles failing
   - Extreme price deviations
   - Stale price handling
   - Network congestion impacts

## Implementation Checklist

- [ ] PriceLib.sol - Price utilities
- [ ] StatisticsLib.sol - Statistical calculations
- [ ] OracleAggregator.sol - Main aggregation logic
- [ ] OracleValidator.sol - Validation logic
- [ ] ChainlinkAdapter.sol - Chainlink integration
- [ ] PythAdapter.sol - Pyth integration
- [ ] FallbackOracle.sol - Emergency prices
- [ ] Unit tests for each component
- [ ] Integration tests
- [ ] Gas optimization
- [ ] Security review

## Dependencies

- External oracle contracts (Chainlink, Pyth)
- Market Registry (for market configurations)

## Estimated Implementation Time

- Core Implementation: 15-20 hours
- Testing: 8-12 hours
- Optimization: 5-8 hours
- Total: 28-40 hours