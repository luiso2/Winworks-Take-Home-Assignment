# Kalshi Market Analyzer

A command-line tool that pulls live prediction market data from Kalshi's public API, identifies markets expiring within configurable time windows, and flags markets with wide bid-ask spreads that may indicate liquidity issues or potential opportunities.

## What It Does

1. **Fetches Live Market Data**: Connects to Kalshi's public API to pull real-time prediction market information
2. **Filters by Expiration**: Identifies markets expiring within 24 hours (configurable)
3. **Analyzes Bid-Ask Spreads**: Calculates spreads and flags markets with spreads > 10%
4. **Displays Key Metrics**: Shows yes/no prices, volume, time to expiration
5. **Ranks by Volume**: Highlights the most actively traded markets

## Quick Start

```bash
# Clone and enter directory
cd winworks-assignment

# Create and activate virtual environment
python3 -m venv venv
source venv/bin/activate

# Install dependencies
pip install -r requirements.txt

# Run the analyzer
python src/kalshi_analyzer.py
```

## Usage

```bash
# Basic usage (24-hour window)
python src/kalshi_analyzer.py

# Custom time window (48 hours)
python src/kalshi_analyzer.py --hours 48

# Fetch more markets
python src/kalshi_analyzer.py --limit 1000

# Filter by minimum volume
python src/kalshi_analyzer.py --min-volume 100
```

### Command Line Options

| Option | Default | Description |
|--------|---------|-------------|
| `--hours` | 24 | Time window for expiring markets |
| `--limit` | 500 | Maximum markets to fetch from API |
| `--min-volume` | 0 | Filter markets by minimum volume |

## Output Sections

### 1. Markets Closing Soon
Shows markets expiring within the specified time window with:
- Market title/description
- Yes/No prices (in cents)
- Bid-ask spread percentage
- Time until expiration
- Trading volume

### 2. Wide Spread Alert
Highlights markets with bid-ask spread > 10%, sorted by spread size. These may indicate:
- Low liquidity
- Price uncertainty
- Potential arbitrage opportunities

### 3. Top Volume Markets
The 10 most actively traded markets, useful for identifying popular/liquid markets.

### 4. Summary Statistics
- Total markets analyzed
- Markets by time window (24h, 48h, 7d)
- Average spread
- Total trading volume

## Technical Details

### API Used
- **Endpoint**: `https://api.elections.kalshi.com/trade-api/v2/markets`
- **Authentication**: None required (public API)
- **Rate Limits**: Not documented, but the tool uses single requests with configurable limits

### Key Fields
- `expected_expiration_time`: When the market resolves (used for filtering)
- `yes_bid` / `yes_ask`: Current bid/ask prices for YES shares (in cents)
- `no_bid` / `no_ask`: Current bid/ask prices for NO shares (in cents)
- `volume`: Total contracts traded

### Spread Calculation
```python
spread = yes_ask - yes_bid
spread_percent = spread * 100
is_wide_spread = spread >= 0.10  # 10% threshold
```

## Dependencies

- `httpx` - HTTP client for API requests
- `rich` - Terminal formatting and tables
- `pendulum` - Date/time handling
- `python-dotenv` - Environment configuration (optional)

## Limitations

1. **Public API Only**: Uses Kalshi's public API which may have rate limits
2. **No Authentication**: Cannot access private/user-specific data
3. **Snapshot Data**: Shows point-in-time data, not real-time streaming
4. **Market Types**: Currently fetches all open markets; could be filtered by category

## Possible Enhancements

- Add real-time streaming via WebSocket
- Support for authenticated API (trading, portfolio)
- Export to CSV/JSON
- Historical spread analysis
- Alerts/notifications for spread opportunities
- Category filtering (politics, sports, economics)

## Author

Jose Hernandez
Winworks Take-Home Assignment
Built with Claude Code

## License

MIT
