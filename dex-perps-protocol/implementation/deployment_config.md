# Deployment Configuration - DEX Perpetual Contracts Protocol

## Overview

This document provides deployment configurations, scripts, and procedures for deploying the DEX Perpetual Contracts Protocol to various networks.

## Network Configurations

### Polygon Mainnet
```javascript
const POLYGON_CONFIG = {
    chainId: 137,
    rpcUrl: "https://polygon-rpc.com",
    explorerUrl: "https://polygonscan.com",
    nativeCurrency: "MATIC",
    blockTime: 2, // seconds
    gasPrice: 30000000000, // 30 gwei
    confirmations: 5,
    contracts: {
        USDC: "0x2791Bca1f2de4661ED88A30C99A7a9449Aa84174",
        USDT: "0xc2132D05D31c914a87C6611C10748AEb04B58e8F",
        DAI: "0x8f3Cf7ad23Cd3CaDbD9735AFf958023239c6A063",
        chainlinkETHUSD: "0xF9680D99D6C9589e2a93a78A04A279e509205945",
        chainlinkBTCUSD: "0xc907E116054Ad103354f2D350FD2514433D57F6f"
    }
};

### Polygon Mumbai Testnet
const MUMBAI_CONFIG = {
    chainId: 80001,
    rpcUrl: "https://rpc-mumbai.maticvigil.com",
    explorerUrl: "https://mumbai.polygonscan.com",
    nativeCurrency: "MATIC",
    blockTime: 2,
    gasPrice: 30000000000,
    confirmations: 3,
    contracts: {
        USDC: "0x0000000000000000000000000000000000000000", // Deploy mock
        USDT: "0x0000000000000000000000000000000000000000", // Deploy mock
        DAI: "0x0000000000000000000000000000000000000000", // Deploy mock
        chainlinkETHUSD: "0x0715A7794a1dc8e42615F059dD6e406A6594651A",
        chainlinkBTCUSD: "0x007A22900a3B98143368Bd5906f8E17e9867581b"
    }
};
```

## Deployment Scripts

### 1. Complete Deployment Script

```javascript
// scripts/deploy-all.js
const hre = require("hardhat");
const { ethers } = require("hardhat");

async function main() {
    console.log("Starting DEX Perpetual Contracts Protocol deployment...");
    
    const [deployer] = await ethers.getSigners();
    console.log("Deploying with account:", deployer.address);
    
    const deploymentConfig = {
        addresses: {},
        transactions: []
    };
    
    // 1. Deploy Market Registry
    console.log("\n1. Deploying Market Registry...");
    const MarketRegistry = await ethers.getContractFactory("MarketRegistry");
    const marketRegistry = await MarketRegistry.deploy();
    await marketRegistry.deployed();
    deploymentConfig.addresses.marketRegistry = marketRegistry.address;
    console.log("Market Registry deployed to:", marketRegistry.address);
    
    // 2. Deploy Access Controller
    console.log("\n2. Deploying Access Controller...");
    const AccessController = await ethers.getContractFactory("AccessController");
    const accessController = await AccessController.deploy();
    await accessController.deployed();
    deploymentConfig.addresses.accessController = accessController.address;
    
    // 3. Deploy Oracle Aggregator
    console.log("\n3. Deploying Oracle Aggregator...");
    const OracleAggregator = await ethers.getContractFactory("OracleAggregator");
    const oracleAggregator = await OracleAggregator.deploy(
        marketRegistry.address,
        accessController.address
    );
    await oracleAggregator.deployed();
    deploymentConfig.addresses.oracleAggregator = oracleAggregator.address;
    
    // 4. Deploy Trading Engine
    console.log("\n4. Deploying Trading Engine...");
    const TradingEngine = await ethers.getContractFactory("TradingEngine");
    const tradingEngine = await TradingEngine.deploy(
        marketRegistry.address,
        oracleAggregator.address,
        accessController.address
    );
    await tradingEngine.deployed();
    deploymentConfig.addresses.tradingEngine = tradingEngine.address;
    
    // 5. Deploy Core Protocol
    console.log("\n5. Deploying Core Protocol...");
    const CoreProtocol = await ethers.getContractFactory("CoreProtocol");
    const coreProtocol = await CoreProtocol.deploy(
        marketRegistry.address,
        oracleAggregator.address,
        tradingEngine.address,
        accessController.address
    );
    await coreProtocol.deployed();
    deploymentConfig.addresses.coreProtocol = coreProtocol.address;
    
    // 6. Deploy Insurance Fund
    console.log("\n6. Deploying Insurance Fund...");
    const InsuranceFund = await ethers.getContractFactory("InsuranceFund");
    const insuranceFund = await InsuranceFund.deploy(
        marketRegistry.address,
        coreProtocol.address,
        accessController.address
    );
    await insuranceFund.deployed();
    deploymentConfig.addresses.insuranceFund = insuranceFund.address;
    
    // 7. Deploy Risk Management
    console.log("\n7. Deploying Risk Management...");
    const RiskManagement = await ethers.getContractFactory("RiskManagement");
    const riskManagement = await RiskManagement.deploy(
        marketRegistry.address,
        coreProtocol.address,
        tradingEngine.address,
        insuranceFund.address,
        oracleAggregator.address,
        accessController.address
    );
    await riskManagement.deployed();
    deploymentConfig.addresses.riskManagement = riskManagement.address;
    
    // 8. Configure cross-contract permissions
    console.log("\n8. Configuring permissions...");
    await configurePermissions(deploymentConfig.addresses);
    
    // 9. Initialize system
    console.log("\n9. Initializing system...");
    await initializeSystem(deploymentConfig.addresses);
    
    // Save deployment configuration
    const fs = require("fs");
    fs.writeFileSync(
        `./deployments/${network.name}-deployment.json`,
        JSON.stringify(deploymentConfig, null, 2)
    );
    
    console.log("\nDeployment completed successfully!");
    console.log("Configuration saved to:", `./deployments/${network.name}-deployment.json`);
}

async function configurePermissions(addresses) {
    const accessController = await ethers.getContractAt("AccessController", addresses.accessController);
    
    // Grant roles to contracts
    const OPERATOR_ROLE = ethers.utils.keccak256(ethers.utils.toUtf8Bytes("OPERATOR"));
    const LIQUIDATOR_ROLE = ethers.utils.keccak256(ethers.utils.toUtf8Bytes("LIQUIDATOR"));
    
    // Trading Engine permissions
    await accessController.grantRole(OPERATOR_ROLE, addresses.tradingEngine);
    console.log("Granted OPERATOR role to Trading Engine");
    
    // Core Protocol permissions
    await accessController.grantRole(OPERATOR_ROLE, addresses.coreProtocol);
    console.log("Granted OPERATOR role to Core Protocol");
    
    // Risk Management permissions
    await accessController.grantRole(LIQUIDATOR_ROLE, addresses.riskManagement);
    console.log("Granted LIQUIDATOR role to Risk Management");
}

async function initializeSystem(addresses) {
    const marketRegistry = await ethers.getContractAt("MarketRegistry", addresses.marketRegistry);
    const oracleAggregator = await ethers.getContractAt("OracleAggregator", addresses.oracleAggregator);
    
    // Set contract addresses in registry
    await marketRegistry.setProtocolContracts({
        tradingEngine: addresses.tradingEngine,
        coreProtocol: addresses.coreProtocol,
        riskManagement: addresses.riskManagement,
        insuranceFund: addresses.insuranceFund,
        oracleAggregator: addresses.oracleAggregator
    });
    
    console.log("Protocol contracts registered");
}

main()
    .then(() => process.exit(0))
    .catch((error) => {
        console.error(error);
        process.exit(1);
    });
```

### 2. Market Creation Script

```javascript
// scripts/create-markets.js
async function createMarkets() {
    const deployment = require(`../deployments/${network.name}-deployment.json`);
    const marketRegistry = await ethers.getContractAt(
        "MarketRegistry",
        deployment.addresses.marketRegistry
    );
    
    // BTC-PERP Market
    const btcMarketConfig = {
        symbol: "BTC-PERP",
        name: "Bitcoin Perpetual",
        baseAsset: ethers.constants.AddressZero,
        tickSize: ethers.utils.parseUnits("0.01", 18), // $0.01
        lotSize: ethers.utils.parseUnits("0.001", 18), // 0.001 BTC
        maxLeverage: 100,
        initialMarginRate: 100, // 1%
        maintenanceMarginRate: 60, // 0.6%
        maxPositionSize: ethers.utils.parseUnits("1000", 18),
        maxOrderSize: ethers.utils.parseUnits("100", 18),
        maxOpenInterest: ethers.utils.parseUnits("10000000", 18), // $10M
        makerFee: -5, // -0.05% rebate
        takerFee: 10, // 0.1%
        liquidationPenalty: 150, // 1.5%
        oraclePriceId: ethers.utils.formatBytes32String("BTC/USD"),
        maxOracleDeviation: 500 // 5%
    };
    
    console.log("Creating BTC-PERP market...");
    const tx1 = await marketRegistry.createMarket(
        btcMarketConfig.symbol,
        btcMarketConfig.name,
        btcMarketConfig
    );
    const receipt1 = await tx1.wait();
    const btcMarketAddress = receipt1.events[0].args.marketId;
    console.log("BTC-PERP created at:", btcMarketAddress);
    
    // ETH-PERP Market
    const ethMarketConfig = {
        ...btcMarketConfig,
        symbol: "ETH-PERP",
        name: "Ethereum Perpetual",
        lotSize: ethers.utils.parseUnits("0.01", 18), // 0.01 ETH
        oraclePriceId: ethers.utils.formatBytes32String("ETH/USD")
    };
    
    console.log("Creating ETH-PERP market...");
    const tx2 = await marketRegistry.createMarket(
        ethMarketConfig.symbol,
        ethMarketConfig.name,
        ethMarketConfig
    );
    const receipt2 = await tx2.wait();
    const ethMarketAddress = receipt2.events[0].args.marketId;
    console.log("ETH-PERP created at:", ethMarketAddress);
    
    // Configure oracles for markets
    await configureMarketOracles(btcMarketAddress, ethMarketAddress);
}

async function configureMarketOracles(btcMarket, ethMarket) {
    const deployment = require(`../deployments/${network.name}-deployment.json`);
    const oracleAggregator = await ethers.getContractAt(
        "OracleAggregator",
        deployment.addresses.oracleAggregator
    );
    
    const networkConfig = network.name === "polygon" ? POLYGON_CONFIG : MUMBAI_CONFIG;
    
    // Configure BTC oracle
    console.log("Configuring BTC oracle...");
    await oracleAggregator.addOracle(
        btcMarket,
        networkConfig.contracts.chainlinkBTCUSD,
        10000, // 100% weight for single oracle
        0 // CHAINLINK type
    );
    
    // Configure ETH oracle
    console.log("Configuring ETH oracle...");
    await oracleAggregator.addOracle(
        ethMarket,
        networkConfig.contracts.chainlinkETHUSD,
        10000,
        0
    );
    
    console.log("Oracle configuration completed");
}
```

### 3. Upgrade Script (for Mainnet)

```javascript
// scripts/upgrade-contract.js
async function upgradeContract(contractName, newImplementation) {
    const { ethers, upgrades } = require("hardhat");
    
    const deployment = require(`../deployments/${network.name}-deployment.json`);
    const proxyAddress = deployment.addresses[contractName];
    
    console.log(`Upgrading ${contractName} at ${proxyAddress}...`);
    
    const NewImplementation = await ethers.getContractFactory(newImplementation);
    const upgraded = await upgrades.upgradeProxy(proxyAddress, NewImplementation);
    
    console.log(`${contractName} upgraded successfully`);
    console.log("New implementation:", await upgrades.erc1967.getImplementationAddress(proxyAddress));
    
    // Update deployment config
    deployment.addresses[`${contractName}Implementation`] = 
        await upgrades.erc1967.getImplementationAddress(proxyAddress);
    
    const fs = require("fs");
    fs.writeFileSync(
        `./deployments/${network.name}-deployment.json`,
        JSON.stringify(deployment, null, 2)
    );
}
```

## Configuration Files

### 1. Initial Parameters (mainnet-params.json)

```json
{
    "markets": {
        "BTC-PERP": {
            "maxLeverage": 50,
            "initialMarginRate": 200,
            "maintenanceMarginRate": 100,
            "maxPositionSize": "500",
            "fundingRateMultiplier": 300,
            "maxFundingRate": 75
        },
        "ETH-PERP": {
            "maxLeverage": 50,
            "initialMarginRate": 200,
            "maintenanceMarginRate": 100,
            "maxPositionSize": "5000",
            "fundingRateMultiplier": 300,
            "maxFundingRate": 75
        }
    },
    "riskParameters": {
        "partialLiquidationThreshold": "50000",
        "partialLiquidationRatio": 5000,
        "liquidationPenalty": 150,
        "priceDeviationShort": 500,
        "priceDeviationMedium": 1000,
        "priceDeviationLong": 2000
    },
    "insuranceFund": {
        "targetRatio": 500,
        "maxUtilization": 8000,
        "tradingFeeAllocation": 4000,
        "treasuryAllocation": 4000,
        "stakingAllocation": 2000
    }
}
```

### 2. Hardhat Configuration

```javascript
// hardhat.config.js
require("@nomiclabs/hardhat-waffle");
require("@nomiclabs/hardhat-etherscan");
require("@openzeppelin/hardhat-upgrades");
require("hardhat-gas-reporter");
require("solidity-coverage");

module.exports = {
    solidity: {
        version: "0.8.19",
        settings: {
            optimizer: {
                enabled: true,
                runs: 200
            }
        }
    },
    networks: {
        hardhat: {
            forking: {
                url: process.env.POLYGON_RPC_URL,
                blockNumber: 35000000
            }
        },
        mumbai: {
            url: process.env.MUMBAI_RPC_URL,
            accounts: [process.env.DEPLOYER_PRIVATE_KEY],
            gasPrice: 30000000000,
            confirmations: 3
        },
        polygon: {
            url: process.env.POLYGON_RPC_URL,
            accounts: [process.env.DEPLOYER_PRIVATE_KEY],
            gasPrice: 50000000000,
            confirmations: 5
        }
    },
    gasReporter: {
        enabled: process.env.REPORT_GAS === "true",
        currency: "USD",
        gasPrice: 30
    },
    etherscan: {
        apiKey: {
            polygon: process.env.POLYGONSCAN_API_KEY,
            polygonMumbai: process.env.POLYGONSCAN_API_KEY
        }
    }
};
```

## Deployment Checklist

### Pre-Deployment
- [ ] All contracts compiled successfully
- [ ] All tests passing (100% coverage)
- [ ] Security audit completed
- [ ] Gas optimization completed
- [ ] Deployment scripts tested on local fork
- [ ] Environment variables configured
- [ ] Multi-sig wallets prepared

### Testnet Deployment
- [ ] Deploy all contracts to Mumbai
- [ ] Verify contracts on Polygonscan
- [ ] Configure initial parameters
- [ ] Create test markets
- [ ] Fund insurance fund
- [ ] Run integration tests
- [ ] Monitor for 48 hours

### Mainnet Deployment
- [ ] Deploy proxy contracts
- [ ] Deploy implementation contracts
- [ ] Initialize proxies
- [ ] Transfer ownership to multi-sig
- [ ] Configure conservative parameters
- [ ] Create initial markets
- [ ] Seed insurance fund
- [ ] Enable monitoring

### Post-Deployment
- [ ] Monitor system metrics
- [ ] Gradual parameter relaxation
- [ ] Liquidity provider onboarding
- [ ] Public announcement
- [ ] Bug bounty program launch

## Emergency Procedures

### Market Freeze
```javascript
// In case of emergency
async function freezeMarket(marketAddress) {
    const marketRegistry = await ethers.getContractAt("MarketRegistry", REGISTRY_ADDRESS);
    const tx = await marketRegistry.freezeMarket(marketAddress);
    await tx.wait();
    console.log("Market frozen:", marketAddress);
}
```

### Emergency Settlement
```javascript
// Force settle all positions at oracle price
async function emergencySettle(marketAddress) {
    const riskManagement = await ethers.getContractAt("RiskManagement", RISK_ADDRESS);
    const tx = await riskManagement.emergencySettleMarket(marketAddress);
    await tx.wait();
    console.log("Emergency settlement completed");
}
```

### System Pause
```javascript
// Pause entire protocol
async function pauseProtocol() {
    const marketRegistry = await ethers.getContractAt("MarketRegistry", REGISTRY_ADDRESS);
    const markets = await marketRegistry.getAllMarkets();
    
    for (const market of markets) {
        await marketRegistry.setMarketStatus(market, 1); // SUSPENDED
    }
    console.log("All markets suspended");
}
```

This deployment configuration provides all necessary scripts and configurations for deploying the DEX Perpetual Contracts Protocol across different networks with proper safety measures.