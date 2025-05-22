# Trading Engine Module Specification

## Overview
The Trading Engine Module handles all order-related operations including placement, validation, matching, and execution. This module is designed as a complete, self-contained unit optimized for AI implementation.

## Component Structure

```
TradingEngine/
├── contracts/
│   ├── OrderBook.sol         // Order storage and book management
│   ├── MatchingEngine.sol    // Order matching algorithm
│   ├── OrderValidator.sol    // Order validation logic
│   └── TradingEvents.sol     // Event definitions
├── interfaces/
│   ├── ITradingEngine.sol    // Main interface
│   └── IOrderBook.sol        // Order book interface
└── libraries/
    ├── OrderLib.sol          // Order data structures
    └── PriceLib.sol          // Price manipulation utilities
```

## Technical Specifications

### Data Structures

```solidity
// Order representation
struct Order {
    uint256 orderId;
    address trader;
    address market;
    OrderType orderType;      // MARKET, LIMIT, POST_ONLY
    Side side;                // BUY, SELL
    uint256 price;            // In base units (e.g., cents for USD)
    uint256 size;             // Contract size
    uint256 filled;           // Amount already filled
    uint256 remainingMargin;  // Margin reserved for order
    uint256 timestamp;
    OrderStatus status;       // ACTIVE, FILLED, CANCELLED
    TimeInForce timeInForce;  // GTC, IOC, FOK
}

// Order book structure
struct OrderBookLevel {
    uint256 price;
    uint256 totalSize;
    DoublyLinkedList.List orders;  // Orders at this price level
}

struct OrderBook {
    // Bid levels sorted high to low
    RedBlackTree.Tree bidTree;
    mapping(uint256 => OrderBookLevel) bidLevels;
    
    // Ask levels sorted low to high
    RedBlackTree.Tree askTree;
    mapping(uint256 => OrderBookLevel) askLevels;
    
    // Order tracking
    mapping(uint256 => Order) orders;
    mapping(address => EnumerableSet.UintSet) userOrderIds;
    
    // Book stats
    uint256 bestBid;
    uint256 bestAsk;
    uint256 lastTradePrice;
}
```

### Core Functions

#### Order Placement
```solidity
function placeOrder(
    address market,
    OrderType orderType,
    Side side,
    uint256 price,
    uint256 size,
    TimeInForce timeInForce
) external returns (uint256 orderId) {
    // 1. Validate order parameters
    _validateOrder(market, orderType, side, price, size);
    
    // 2. Check margin requirements
    uint256 requiredMargin = _calculateRequiredMargin(market, size, price);
    _checkAndReserveMargin(msg.sender, requiredMargin);
    
    // 3. Create order object
    orderId = _generateOrderId();
    Order memory order = Order({
        orderId: orderId,
        trader: msg.sender,
        market: market,
        orderType: orderType,
        side: side,
        price: price,
        size: size,
        filled: 0,
        remainingMargin: requiredMargin,
        timestamp: block.timestamp,
        status: OrderStatus.ACTIVE,
        timeInForce: timeInForce
    });
    
    // 4. Attempt immediate execution
    if (orderType == OrderType.MARKET || _canExecuteImmediately(order)) {
        _executeOrder(order);
    }
    
    // 5. Add to book if not fully filled
    if (order.filled < order.size && timeInForce == TimeInForce.GTC) {
        _addToOrderBook(order);
    }
    
    emit OrderPlaced(orderId, msg.sender, market, order);
}
```

#### Order Matching
```solidity
function matchOrders(address market, uint256 maxIterations) external {
    OrderBook storage book = orderBooks[market];
    uint256 iterations = 0;
    
    while (iterations < maxIterations) {
        // Check if matching is possible
        if (book.bestBid == 0 || book.bestAsk == 0 || book.bestBid < book.bestAsk) {
            break;
        }
        
        // Get top orders
        Order storage bidOrder = _getTopBidOrder(book);
        Order storage askOrder = _getTopAskOrder(book);
        
        // Calculate match size
        uint256 matchSize = Math.min(
            bidOrder.size - bidOrder.filled,
            askOrder.size - askOrder.filled
        );
        
        // Execute match
        uint256 matchPrice = _determineMatchPrice(bidOrder, askOrder);
        _executeTrade(bidOrder, askOrder, matchSize, matchPrice);
        
        // Update orders
        bidOrder.filled += matchSize;
        askOrder.filled += matchSize;
        
        // Remove filled orders
        if (bidOrder.filled == bidOrder.size) {
            _removeFromOrderBook(book, bidOrder);
        }
        if (askOrder.filled == askOrder.size) {
            _removeFromOrderBook(book, askOrder);
        }
        
        iterations++;
    }
}
```

### Gas Optimization Techniques

1. **Lazy Deletion**: Mark orders as cancelled rather than deleting
2. **Batch Processing**: Match multiple orders per transaction
3. **Price Level Caching**: Store best bid/ask for quick access
4. **Packed Storage**: Combine related fields in single storage slot

### Security Measures

1. **Reentrancy Protection**: NonReentrant modifier on all external functions
2. **Overflow Protection**: Using Solidity 0.8+ automatic checks
3. **Access Control**: Order cancellation only by owner
4. **Validation**: Comprehensive input validation

### Testing Requirements

1. **Unit Tests**:
   - Order placement with various parameters
   - Order matching scenarios
   - Edge cases (empty book, self-trade, etc.)

2. **Integration Tests**:
   - Interaction with Core Protocol
   - Margin reservation and release
   - Event emission verification

3. **Stress Tests**:
   - High-volume order placement
   - Large order book management
   - Gas consumption analysis

## Implementation Checklist

- [ ] OrderLib.sol - Data structures and utilities
- [ ] OrderBook.sol - Order book management
- [ ] OrderValidator.sol - Validation logic
- [ ] MatchingEngine.sol - Matching algorithm
- [ ] TradingEvents.sol - Event definitions
- [ ] ITradingEngine.sol - Interface definition
- [ ] Unit tests for each component
- [ ] Integration tests
- [ ] Gas optimization pass
- [ ] Security review

## Dependencies

- Core Protocol Engine (for position updates)
- Market Registry (for market parameters)
- Oracle Module (for price validation)

## Estimated Implementation Time

- Core Implementation: 20-25 hours
- Testing: 10-15 hours
- Optimization: 5-10 hours
- Total: 35-50 hours