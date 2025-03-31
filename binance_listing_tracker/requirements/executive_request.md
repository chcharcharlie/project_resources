# Binance Listing Tracker - Executive Request

## Product Overview
An automated system that monitors Binance announcements for new coin listings, gathers key data about these assets, analyzes their investment potential, and sends notifications via Telegram. The system will check for new listings every 2 minutes, evaluate each listing based on configured criteria, and provide fact-based summaries to aid investment decisions.

## Business Objectives
- Identify potentially profitable investment opportunities from new Binance listings quickly
- Save time by automating the research process for new crypto assets
- Provide objective data-driven analysis to support investment decisions
- Reduce the risk of missing early investment opportunities due to delayed awareness

## Target Users
- Individual crypto investors (primarily the system owner)
- Focus on users who seek early information on new listings but lack time to constantly monitor announcements

## Core Functionality
1. **Automated Monitoring**
   - Poll the Binance announcements page (https://www.binance.com/en/support/announcement/list/48) every 2 minutes
   - Detect and parse new coin listing announcements
   - Extract key information from the announcement text

2. **Data Collection & Analysis**
   - Gather financial metrics (particularly FDV - Fully Diluted Valuation)
   - Research project background, investors, funding history
   - Analyze the project's narrative, utility, and innovation potential
   - Use AI to evaluate the data and generate a summary

3. **Investment Evaluation**
   - Apply configurable criteria to assess investment potential
   - Initial FDV threshold set at 150M (configurable parameter)
   - Summarize the project's narrative and key differentiators
   - Present facts in an organized, easily digestible format

4. **Notification System**
   - Send alerts via Telegram when new listings are detected
   - Include comprehensive analysis in the notification
   - Format message for optimal readability on mobile devices

5. **Future Expansion Capability**
   - Design with the ability to add automated trading functionality later
   - Allow for adding additional notification channels (e.g., Slack)
   - Support customizable analysis parameters and thresholds

## Technical Requirements
- Ability to scrape web content from Binance announcements
- Integration with financial data APIs to gather metrics
- AI-powered analysis using LLMs and potentially MCPs for tool usage
- Telegram API integration for notifications
- Persistent storage to track previously analyzed listings
- Configurable parameters for investment thresholds

## Constraints
- System must be reliable enough to run continuously without frequent maintenance
- Analysis should complete quickly enough to provide timely notification
- Must handle rate limits of any external APIs appropriately
- System should be cost-effective to operate

## Success Metrics
- Successful detection of 100% of new coin listings
- Accuracy of gathered financial and project data
- Timeliness of notifications (within minutes of official announcement)
- Quality and relevance of the analysis provided
- User satisfaction with the notification content and format
