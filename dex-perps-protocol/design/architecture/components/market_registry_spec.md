# Market Registry Module Specification

## Overview
The Market Registry Module manages market configurations, parameters, and access control for the entire protocol. It serves as the central configuration hub that all other modules reference for market-specific settings.

## Component Structure

```
MarketRegistry/
├── contracts/
│   ├── MarketRegistry.sol       // Main registry contract
│   ├── ParameterStore.sol       // Configuration storage
│   ├── AccessController.sol     // Role-based access control
│   └── EmergencyAdmin.sol       // Emergency controls
├── interfaces/
│   ├── IMarketRegistry.sol      // Main interface
│   └── IAccessController.sol    // Access control interface
└── libraries/
    ├── MarketLib.sol            // Market data structures
    └── ValidationLib.sol        // Parameter validation
```

## Technical Specifications

### Data Structures

```solidity
// Complete market configuration
struct MarketConfig {
    // Basic information
    string symbol;                    // e.g., "AAPL-PERP"
    string name;                      // e.g., "Apple Inc. Perpetual"
    address baseAsset;                // Underlying asset identifier
    
    // Trading parameters
    uint256 tickSize;                 // Minimum price increment (e.g., 0.01 USD)
    uint256 lotSize;                  // Minimum order size (e.g., 0.001 contracts)
    uint256 maxLeverage;              // Maximum allowed leverage (e.g., 100)
    
    // Risk parameters
    uint256 initialMarginRate;        // Initial margin requirement (basis points)
    uint256 maintenanceMarginRate;    // Maintenance margin requirement (basis points)
    uint256 maxPositionSize;          // Max position per trader
    uint256 maxOrderSize;             // Max size per order
    uint256 maxOpenInterest;          // Max total open interest
    
    // Fee structure
    int256 makerFee;                  // Can be negative for rebates (basis points)
    uint256 takerFee;                 // Taker fee (basis points)
    uint256 liquidationPenalty;       // Liquidation penalty (basis points)
    
    // Oracle configuration
    bytes32 oraclePriceId;            // Price feed identifier
    uint256 maxOracleDeviation;       // Max acceptable oracle price deviation
    
    // Market status
    MarketStatus status;              // ACTIVE, SUSPENDED, CLOSE_ONLY
    bool isFrozen;                    // Emergency freeze flag
    uint256 createdAt;                // Market creation timestamp
    uint256 lastUpdated;              // Last configuration update
}

// Market status enumeration
enum MarketStatus {
    ACTIVE,          // Normal trading
    SUSPENDED,       // Temporarily suspended
    CLOSE_ONLY,      // Only position closing allowed
    SETTLEMENT       // Final settlement mode
}

// Dynamic market parameters (can be updated more frequently)
struct DynamicParameters {
    uint256 fundingRateMultiplier;   // Funding rate adjustment factor
    uint256 maxFundingRate;           // Maximum funding rate cap
    uint256 liquidationBufferRatio;  // Extra margin buffer for liquidations
    uint256 adlThreshold;            // Auto-deleveraging trigger threshold
}

// Role definitions
struct RoleConfig {
    bytes32 role;
    string description;
    address[] members;
    uint256 memberCount;
}

// Parameter update request (for timelock)
struct ParameterUpdate {
    address market;
    bytes32 parameter;
    uint256 oldValue;
    uint256 newValue;
    uint256 scheduledTime;
    bool executed;
}
```

### Core Functions

#### Market Management
```solidity
function createMarket(
    string memory symbol,
    string memory name,
    MarketConfig memory config
) external onlyAdmin returns (address marketId) {
    // 1. Validate market configuration
    _validateMarketConfig(config);
    require(marketsBySymbol[symbol] == address(0), "Market already exists");
    
    // 2. Generate market ID
    marketId = address(uint160(uint256(keccak256(abi.encodePacked(
        symbol,
        block.timestamp,
        msg.sender
    )))));
    
    // 3. Store market configuration
    markets[marketId] = config;
    markets[marketId].symbol = symbol;
    markets[marketId].name = name;
    markets[marketId].createdAt = block.timestamp;
    markets[marketId].lastUpdated = block.timestamp;
    markets[marketId].status = MarketStatus.ACTIVE;
    
    // 4. Initialize dynamic parameters with defaults
    dynamicParameters[marketId] = DynamicParameters({
        fundingRateMultiplier: 300,    // 3x multiplier (basis points)
        maxFundingRate: 100,            // 1% max funding rate
        liquidationBufferRatio: 50,     // 0.5% buffer
        adlThreshold: 8000              // 80% fund utilization triggers ADL
    });
    
    // 5. Set up market indices
    marketsBySymbol[symbol] = marketId;
    marketList.push(marketId);
    activeMarketCount++;
    
    // 6. Initialize market in other modules
    _initializeMarketModules(marketId, config);
    
    emit MarketCreated(marketId, symbol, config);
}

function updateMarketConfig(
    address market,
    bytes32 parameter,
    uint256 newValue
) external onlyOperator {
    require(markets[market].createdAt > 0, "Market does not exist");
    require(!markets[market].isFrozen, "Market is frozen");
    
    // 1. Get current value
    uint256 currentValue = _getParameterValue(market, parameter);
    require(newValue != currentValue, "Value unchanged");
    
    // 2. Validate parameter change
    _validateParameterUpdate(market, parameter, currentValue, newValue);
    
    // 3. Check if parameter requires timelock
    if (_requiresTimelock(parameter)) {
        // Schedule update for timelock
        bytes32 updateId = keccak256(abi.encodePacked(
            market,
            parameter,
            newValue,
            block.timestamp
        ));
        
        parameterUpdates[updateId] = ParameterUpdate({
            market: market,
            parameter: parameter,
            oldValue: currentValue,
            newValue: newValue,
            scheduledTime: block.timestamp + TIMELOCK_DELAY,
            executed: false
        });
        
        emit ParameterUpdateScheduled(market, parameter, currentValue, newValue, updateId);
    } else {
        // Immediate update for non-critical parameters
        _updateParameter(market, parameter, newValue);
        emit ParameterUpdated(market, parameter, currentValue, newValue);
    }
}

function executeScheduledUpdate(bytes32 updateId) external {
    ParameterUpdate storage update = parameterUpdates[updateId];
    require(update.scheduledTime > 0, "Update not found");
    require(!update.executed, "Already executed");
    require(block.timestamp >= update.scheduledTime, "Timelock not expired");
    
    // Execute the update
    _updateParameter(update.market, update.parameter, update.newValue);
    update.executed = true;
    
    emit ParameterUpdateExecuted(
        update.market,
        update.parameter,
        update.oldValue,
        update.newValue
    );
}

function _updateParameter(
    address market,
    bytes32 parameter,
    uint256 newValue
) internal {
    MarketConfig storage config = markets[market];
    
    if (parameter == "maxLeverage") {
        config.maxLeverage = newValue;
    } else if (parameter == "initialMarginRate") {
        config.initialMarginRate = newValue;
    } else if (parameter == "maintenanceMarginRate") {
        config.maintenanceMarginRate = newValue;
    } else if (parameter == "makerFee") {
        config.makerFee = int256(newValue);
    } else if (parameter == "takerFee") {
        config.takerFee = newValue;
    } else if (parameter == "maxPositionSize") {
        config.maxPositionSize = newValue;
    } else if (parameter == "maxOrderSize") {
        config.maxOrderSize = newValue;
    } else if (parameter == "fundingRateMultiplier") {
        dynamicParameters[market].fundingRateMultiplier = newValue;
    } else {
        revert("Unknown parameter");
    }
    
    config.lastUpdated = block.timestamp;
}
```

#### Market Status Control
```solidity
function setMarketStatus(
    address market,
    MarketStatus newStatus
) external onlyOperator {
    MarketConfig storage config = markets[market];
    require(config.createdAt > 0, "Market does not exist");
    
    MarketStatus oldStatus = config.status;
    require(oldStatus != newStatus, "Status unchanged");
    
    // Validate status transition
    if (newStatus == MarketStatus.SETTLEMENT) {
        require(
            oldStatus == MarketStatus.CLOSE_ONLY,
            "Must be in CLOSE_ONLY before SETTLEMENT"
        );
    }
    
    config.status = newStatus;
    
    // Execute status-specific actions
    if (newStatus == MarketStatus.SUSPENDED) {
        // Cancel all open orders
        tradingEngine.cancelAllMarketOrders(market);
    } else if (newStatus == MarketStatus.CLOSE_ONLY) {
        // Prevent new position opening
        tradingEngine.setCloseOnlyMode(market, true);
    } else if (newStatus == MarketStatus.SETTLEMENT) {
        // Initiate final settlement process
        _initiateSettlement(market);
    }
    
    emit MarketStatusChanged(market, oldStatus, newStatus);
}

function freezeMarket(address market) external onlyEmergencyAdmin {
    MarketConfig storage config = markets[market];
    require(!config.isFrozen, "Already frozen");
    
    config.isFrozen = true;
    
    // Emergency actions
    tradingEngine.cancelAllMarketOrders(market);
    
    emit MarketFrozen(market, msg.sender);
}

function unfreezeMarket(address market) external onlyAdmin {
    MarketConfig storage config = markets[market];
    require(config.isFrozen, "Not frozen");
    
    config.isFrozen = false;
    
    emit MarketUnfrozen(market, msg.sender);
}
```

#### Access Control
```solidity
contract AccessController {
    // Role definitions
    bytes32 public constant ADMIN_ROLE = keccak256("ADMIN");
    bytes32 public constant OPERATOR_ROLE = keccak256("OPERATOR");
    bytes32 public constant EMERGENCY_ADMIN_ROLE = keccak256("EMERGENCY_ADMIN");
    bytes32 public constant LIQUIDATOR_ROLE = keccak256("LIQUIDATOR");
    bytes32 public constant KEEPER_ROLE = keccak256("KEEPER");
    
    mapping(bytes32 => RoleData) private roles;
    mapping(address => mapping(bytes32 => bool)) private members;
    
    struct RoleData {
        mapping(address => bool) members;
        uint256 memberCount;
        bytes32 adminRole;
    }
    
    function grantRole(bytes32 role, address account) external onlyRoleAdmin(role) {
        require(!hasRole(role, account), "Already has role");
        
        roles[role].members[account] = true;
        roles[role].memberCount++;
        members[account][role] = true;
        
        emit RoleGranted(role, account, msg.sender);
    }
    
    function revokeRole(bytes32 role, address account) external onlyRoleAdmin(role) {
        require(hasRole(role, account), "Does not have role");
        
        roles[role].members[account] = false;
        roles[role].memberCount--;
        members[account][role] = false;
        
        emit RoleRevoked(role, account, msg.sender);
    }
    
    function hasRole(bytes32 role, address account) public view returns (bool) {
        return roles[role].members[account];
    }
    
    function getRoleMemberCount(bytes32 role) external view returns (uint256) {
        return roles[role].memberCount;
    }
    
    modifier onlyRole(bytes32 role) {
        require(hasRole(role, msg.sender), "AccessControl: unauthorized");
        _;
    }
}
```

#### Parameter Validation
```solidity
library ValidationLib {
    function validateMarketConfig(MarketConfig memory config) internal pure {
        // Trading parameters
        require(config.tickSize > 0, "Invalid tick size");
        require(config.lotSize > 0, "Invalid lot size");
        require(config.maxLeverage >= 1 && config.maxLeverage <= 100, "Invalid leverage");
        
        // Margin requirements
        require(
            config.initialMarginRate >= config.maintenanceMarginRate,
            "Initial margin must be >= maintenance"
        );
        require(config.maintenanceMarginRate >= 50, "Maintenance margin too low"); // 0.5% min
        require(config.initialMarginRate <= 10000, "Initial margin too high"); // 100% max
        
        // Position limits
        require(config.maxPositionSize > 0, "Invalid max position");
        require(config.maxOrderSize > 0 && config.maxOrderSize <= config.maxPositionSize, "Invalid max order");
        
        // Fees
        require(config.takerFee <= 1000, "Taker fee too high"); // 10% max
        require(config.makerFee >= -100 && config.makerFee <= 1000, "Invalid maker fee");
        require(config.liquidationPenalty <= 500, "Liquidation penalty too high"); // 5% max
    }
    
    function validateParameterUpdate(
        address market,
        bytes32 parameter,
        uint256 currentValue,
        uint256 newValue
    ) internal view {
        // Prevent extreme parameter changes
        uint256 maxChange = 5000; // 50% max change
        
        if (parameter == "maxLeverage") {
            require(newValue >= 1 && newValue <= 100, "Leverage out of range");
            // Leverage can only be reduced, not increased
            require(newValue <= currentValue, "Cannot increase max leverage");
        } else if (parameter == "maintenanceMarginRate") {
            uint256 change = currentValue > newValue ? 
                currentValue - newValue : newValue - currentValue;
            require(change <= (currentValue * maxChange) / 10000, "Change too large");
        }
        
        // Additional parameter-specific validations...
    }
}
```

#### Emergency Controls
```solidity
contract EmergencyAdmin {
    uint256 public constant EMERGENCY_DELAY = 12 hours;
    
    mapping(bytes32 => EmergencyAction) public emergencyActions;
    
    struct EmergencyAction {
        address target;
        bytes data;
        uint256 scheduledTime;
        bool executed;
        string reason;
    }
    
    function scheduleEmergencyAction(
        address target,
        bytes memory data,
        string memory reason
    ) external onlyRole(EMERGENCY_ADMIN_ROLE) returns (bytes32 actionId) {
        actionId = keccak256(abi.encodePacked(target, data, block.timestamp));
        
        emergencyActions[actionId] = EmergencyAction({
            target: target,
            data: data,
            scheduledTime: block.timestamp + EMERGENCY_DELAY,
            executed: false,
            reason: reason
        });
        
        emit EmergencyActionScheduled(actionId, target, reason);
    }
    
    function executeEmergencyAction(bytes32 actionId) external onlyRole(EMERGENCY_ADMIN_ROLE) {
        EmergencyAction storage action = emergencyActions[actionId];
        require(action.scheduledTime > 0, "Action not found");
        require(!action.executed, "Already executed");
        require(block.timestamp >= action.scheduledTime, "Delay not passed");
        
        action.executed = true;
        
        (bool success, bytes memory result) = action.target.call(action.data);
        require(success, "Emergency action failed");
        
        emit EmergencyActionExecuted(actionId, result);
    }
    
    function cancelEmergencyAction(bytes32 actionId) external onlyRole(ADMIN_ROLE) {
        require(emergencyActions[actionId].scheduledTime > 0, "Action not found");
        require(!emergencyActions[actionId].executed, "Already executed");
        
        delete emergencyActions[actionId];
        
        emit EmergencyActionCancelled(actionId);
    }
}
```

### View Functions
```solidity
function getMarketConfig(address market) external view returns (MarketConfig memory) {
    require(markets[market].createdAt > 0, "Market does not exist");
    return markets[market];
}

function getMarketBySymbol(string memory symbol) external view returns (address) {
    return marketsBySymbol[symbol];
}

function getAllMarkets() external view returns (address[] memory) {
    return marketList;
}

function getActiveMarkets() external view returns (address[] memory) {
    uint256 count = 0;
    for (uint i = 0; i < marketList.length; i++) {
        if (markets[marketList[i]].status == MarketStatus.ACTIVE) {
            count++;
        }
    }
    
    address[] memory active = new address[](count);
    uint256 index = 0;
    
    for (uint i = 0; i < marketList.length; i++) {
        if (markets[marketList[i]].status == MarketStatus.ACTIVE) {
            active[index++] = marketList[i];
        }
    }
    
    return active;
}

function isMarketActive(address market) external view returns (bool) {
    return markets[market].status == MarketStatus.ACTIVE && !markets[market].isFrozen;
}

function canOpenPosition(address market) external view returns (bool) {
    MarketStatus status = markets[market].status;
    return status == MarketStatus.ACTIVE && !markets[market].isFrozen;
}
```

### Security Measures

1. **Role-Based Access**: Granular permissions for different operations
2. **Timelock Mechanism**: Critical parameter changes delayed
3. **Parameter Validation**: Comprehensive checks on all updates
4. **Emergency Controls**: Quick response to critical situations
5. **Audit Trail**: All changes logged with events

### Testing Requirements

1. **Unit Tests**:
   - Market creation and configuration
   - Parameter updates with timelock
   - Access control role management
   - Emergency procedures

2. **Integration Tests**:
   - Market status changes affecting trading
   - Parameter updates affecting other modules
   - Multi-role authorization flows

3. **Edge Cases**:
   - Invalid parameter combinations
   - Status transition validations
   - Concurrent update attempts

## Implementation Checklist

- [ ] MarketLib.sol - Market data structures
- [ ] ValidationLib.sol - Validation utilities
- [ ] MarketRegistry.sol - Main registry logic
- [ ] ParameterStore.sol - Configuration management
- [ ] AccessController.sol - Role management
- [ ] EmergencyAdmin.sol - Emergency controls
- [ ] Unit tests for each component
- [ ] Integration tests
- [ ] Gas optimization
- [ ] Security review

## Dependencies

- Used by all other modules for market configuration
- No dependencies on other modules (base layer)

## Estimated Implementation Time

- Core Implementation: 12-15 hours
- Testing: 6-8 hours
- Optimization: 3-5 hours
- Total: 21-28 hours