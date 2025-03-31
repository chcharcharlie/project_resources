# Binance Listing Tracker - Product Requirements Document

## 1. Product Overview

### 1.1 Product Purpose
The Binance Listing Tracker is an automated system that monitors Binance's announcement page for new coin listings, collects and analyzes relevant data about these assets, and sends notifications with investment insights via Telegram. The system aims to provide timely information about new listing opportunities to support faster, more informed investment decisions.

### 1.2 Target Users
- Individual crypto investors (primarily the system owner)
- Users seeking early information on new listings without constant manual monitoring

### 1.3 Value Proposition
- Eliminates the need for manual monitoring of Binance announcements
- Provides rapid access to key investment data for newly listed coins
- Delivers organized, relevant information to support quick decision-making
- Reduces the risk of missing early investment opportunities

## 2. Feature Specifications

### 2.1 Binance Announcement Monitoring
#### 2.1.1 Polling Mechanism
- System will check the Binance announcement page (https://www.binance.com/en/support/announcement/list/48) every 2 minutes
- Polling interval will be configurable via a configuration file
- System will use appropriate request headers and respect Binance's robots.txt policies
- Implementation will include rate limiting protection and retry logic

#### 2.1.2 Announcement Detection
- System will parse the HTML structure of the announcement page
- System will identify new coin listing announcements by analyzing announcement titles and content
- System will maintain a persistent record of previously processed announcements to avoid duplicate processing
- System will extract the following data from announcements:
  - Coin/token name and ticker symbol
  - Listing date and time
  - Trading pairs being added
  - Any special notes or restrictions mentioned in the announcement

#### 2.1.3 Error Handling
- System will implement exponential backoff for failed connection attempts
- System will log detailed error information
- System will continue operation despite temporary failures
- System will send an admin alert if persistent failures occur (more than 10 consecutive failures)

### 2.2 Data Collection and Research

#### 2.2.1 Data Sources Strategy
The system will use a multi-tiered approach to gather information about newly listed coins:

**Primary Data Sources:**
1. Binance announcement details
2. Coin's official website (extracted from announcement or found via search)
3. CoinGecko, CoinMarketCap API (if the coin is already listed there)
4. Token contract data from blockchain explorers

**Secondary Data Sources (if primary sources insufficient):**
1. GitHub repositories of the project
2. Social media accounts (Twitter/X, Discord, Telegram)
3. Recent news articles about the project
4. Community forums (Reddit, specialized crypto forums)

**Data Retrieval Methods:**
1. Direct API calls (for services with available APIs)
2. Browser automation for data requiring web navigation
3. Blockchain RPC calls for on-chain data
4. AI-assisted extraction from text and web content
5. Search engine queries for supplementary information

#### 2.2.2 Financial Metrics Collection
- System will attempt to gather FDV (Fully Diluted Valuation) using:
  - Market data APIs (CoinGecko, CoinMarketCap) if available
  - Calculation from on-chain data: total supply * current price
  - Information from the project's tokenomics documentation
- System will record current price, 24h volume (if available)
- System will attempt to gather market cap, circulating supply information
- System will note when data is estimated vs. officially confirmed

#### 2.2.3 Project Background Research
- System will collect information about:
  - Project team (founders, key team members)
  - Notable investors and funding rounds
  - Project launch date and development history
  - Partnerships and integrations
- System will prioritize official sources for this information
- System will note when information is from secondary or unofficial sources

#### 2.2.4 Narrative and Utility Analysis
- System will identify the project's:
  - Main category/sector (DeFi, Gaming, Infrastructure, etc.)
  - Primary use case and problem being solved
  - Unique selling points or technological innovations
  - Competitive landscape and differentiation
- System will extract quotes from project documentation about utility and vision
- System will note market trends related to the project's category

### 2.3 Investment Analysis

#### 2.3.1 Data Evaluation Framework
- System will apply configurable criteria to evaluate investment potential
- Initial evaluation will focus on:
  - FDV compared to threshold (initially 150M)
  - Strength of narrative and uniqueness
  - Quality of team and backers
  - Market timing and sector trends
- System will identify red flags such as:
  - Extremely high initial valuation
  - Vague utility or copycat project characteristics
  - Limited information availability
  - Poor tokenomics design

#### 2.3.2 AI-Powered Analysis
- System will use AI to:
  - Summarize key project information
  - Identify strengths and weaknesses
  - Categorize the project based on its characteristics
  - Extract sentiment from available community discussions
- AI will integrate and synthesize data from multiple sources
- Analysis will prioritize factual information over speculative elements
- Analysis will highlight information quality and confidence levels

#### 2.3.3 Investment Summary Generation
- System will generate a structured investment summary including:
  - Key financial metrics with sources
  - Project background highlights
  - Narrative strength assessment
  - Notable strengths and concerns
  - Data confidence level indicators
- Summary will be concise but comprehensive
- Summary will separate facts from analysis-based conclusions

### 2.4 Notification System

#### 2.4.1 Telegram Integration
- System will integrate with Telegram Bot API
- System will send notifications to a predefined chat ID
- Messages will be formatted using Markdown for readability
- System will include a timestamp of when the analysis was performed

#### 2.4.2 Notification Content
- Each notification will include:
  - Announcement title and direct link
  - Basic token information (name, symbol, chains)
  - Key financial metrics (FDV, price, etc.)
  - Concise project summary (2-3 paragraphs)
  - Investment highlights (bullet points)
  - Data confidence assessment
- Notifications will be designed for mobile readability
- Long content will be appropriately segmented

#### 2.4.3 Notification Controls
- System will implement a configuration to enable/disable notifications
- System will support notification filters based on configurable criteria (e.g., only notify for FDV < threshold)
- System will include a notification cooldown to prevent spam in case of multiple listings
- System will support user acknowledgment tracking (future feature)

### 2.5 System Configuration and Management

#### 2.5.1 Configuration Options
- System will support a configuration file with the following parameters:
  - Polling frequency
  - FDV threshold
  - Telegram chat ID and bot token
  - Notification preferences
  - API keys for data services
  - Logging levels
- Configuration will be separated from code for easy updates
- System will validate configuration on startup

#### 2.5.2 Logging and Monitoring
- System will maintain detailed logs of:
  - Polling operations
  - Data collection attempts and results
  - Analysis process
  - Notification delivery status
- Logs will be rotated to manage disk usage
- System will support optional health check endpoints for monitoring

#### 2.5.3 Error Recovery
- System will implement self-healing mechanisms for common errors
- System will maintain state to resume operation after crashes
- System will include a watchdog process for automatic restarts if needed

## 3. User Experience

### 3.1 User Touchpoints
The system is largely automated, with the following user touchpoints:
1. System configuration (initial setup)
2. Telegram notifications (primary user interaction)
3. Logs and monitoring (for troubleshooting)

### 3.2 Notification Experience
- Notifications will arrive promptly after new listing announcements
- Content will be structured for quick scanning and decision-making
- Critical information will be highlighted or emphasized
- Direct links will be provided for further research

### 3.3 User Controls
- User can control system behavior through configuration file updates
- User can silence notifications by configuring the Telegram bot
- User can request system status through predefined commands (future feature)

## 4. Technical Architecture

### 4.1 System Components

#### 4.1.1 Polling Service
- Responsible for checking Binance announcements on schedule
- Detects new listings and extracts initial data
- Maintains record of processed announcements
- Triggers the data collection and analysis pipeline

#### 4.1.2 Data Collection Engine
- Implements various data collection strategies
- Coordinates parallel data gathering from multiple sources
- Handles rate limiting and access patterns for different services
- Normalizes collected data into a standard format

#### 4.1.3 Analysis Engine
- Processes collected data through AI analysis
- Generates investment summaries and recommendations
- Applies configured evaluation criteria
- Produces structured output for notifications

#### 4.1.4 Notification Service
- Formats analysis results for Telegram
- Handles message delivery and retry logic
- Manages notification preferences and filtering
- Tracks delivery status

#### 4.1.5 Persistence Layer
- Stores record of processed announcements
- Maintains collected data and analysis results
- Handles configuration storage
- Manages logging

### 4.2 External Integrations

#### 4.2.1 Binance Website
- Web scraping to extract announcement information
- HTML parsing for structured data extraction
- Rate-limited interaction to prevent IP blocks

#### 4.2.2 Data Services
**Primary integrations:**
- CoinGecko API (if coin is listed)
- CoinMarketCap API (if coin is listed)
- Blockchain explorers (Etherscan, BscScan, etc.)
- Web browsers for data requiring navigation

**Secondary integrations:**
- Search engines for finding supplementary information
- Project websites and documentation
- Social media platforms for community sentiment

#### 4.2.3 Telegram API
- Bot API for sending notifications
- Message formatting and delivery handling
- Error management for failed deliveries

### 4.3 Data Flow
1. Polling Service checks Binance announcements
2. New announcements trigger data collection process
3. Data Collection Engine gathers information from multiple sources
4. Collected data is passed to Analysis Engine
5. Analysis Engine generates investment summary
6. Summary is formatted and sent via Notification Service
7. Process details and results are stored in Persistence Layer

### 4.4 Deployment Considerations
- System designed to run as a background service
- Low resource requirements for continuous operation
- Suitable for deployment on a small VPS or similar environment
- Docker containerization recommended for easy deployment
- Persistent volume needed for state management

## 5. Non-Functional Requirements

### 5.1 Performance
- Polling must complete within the assigned interval (2 minutes)
- Data collection should complete within 5 minutes of announcement detection
- Notification should be delivered within 10 minutes of announcement detection
- System should handle temporary outages of dependent services gracefully

### 5.2 Reliability
- System must achieve 99% uptime
- No announcements should be missed due to system errors
- Data collection failures for one source should not prevent using other sources
- System must recover automatically from common error conditions

### 5.3 Security
- API keys and tokens must be stored securely
- System should not expose internal data or configuration
- Web interaction must follow responsible scraping practices
- Telegram bot token must be protected from unauthorized access

### 5.4 Maintainability
- Code must be well-documented and structured
- Configuration changes should not require code modifications
- Logging must provide sufficient information for troubleshooting
- Components should be modular to allow for easy updates

## 6. Implementation Guidelines

### 6.1 Development Priorities
1. Core announcement monitoring functionality
2. Basic data collection from most reliable sources
3. Simple analysis and notification capabilities
4. Enhanced data collection from secondary sources
5. Advanced AI analysis and recommendations
6. System monitoring and self-healing capabilities

### 6.2 Testing Requirements
- Unit tests for each component
- Integration tests for external service interactions
- End-to-end tests for the complete workflow
- Mocked data for testing without live dependencies
- Performance testing to ensure timely operation

### 6.3 Deployment Considerations
- Environment setup documentation
- Configuration templates
- Docker compose files for easy deployment
- Backup and restore procedures
- Monitoring setup guidance

## 7. Future Enhancements

### 7.1 Short-term Enhancements
- Additional notification channels (Slack, email)
- Enhanced data visualization in notifications
- User command interface for on-demand information
- Performance optimizations for faster analysis

### 7.2 Medium-term Enhancements
- Historical performance tracking of previous listings
- Pattern recognition for successful investments
- Customizable analysis parameters
- Web dashboard for system monitoring and configuration

### 7.3 Long-term Vision
- Automated trading integration based on analysis
- Multi-exchange monitoring beyond Binance
- Advanced market sentiment analysis
- Portfolio impact analysis for investment recommendations
