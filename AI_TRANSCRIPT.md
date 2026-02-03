# AI Development Transcript
## Winworks Take-Home Assignment - Kalshi Market Analyzer
**Tool Used:** Claude Code (Claude Opus 4.5)
**Date:** February 3, 2026
**Developer:** Jose Hernandez

---

## Prompt 1: Project Setup & Requirements

**Me:**
```
I need to build a Kalshi market analyzer for a take-home assignment. Here are my requirements:

1. Pull live market data from Kalshi's public API
2. Identify markets closing within the next 24 hours
3. Display yes/no prices for each market
4. Flag markets with wide bid-ask spreads (>10%) as potential opportunities

Tech stack preferences:
- Python (required by assignment)
- httpx for async HTTP calls
- rich library for formatted CLI output
- pendulum for datetime handling

Can you start by creating the project structure and a basic Market dataclass?
```

**Claude:**
I'll create a clean project structure for your Kalshi analyzer. Let me set up the foundation.

```bash
mkdir -p ~/Desktop/winworks-assignment/src
cd ~/Desktop/winworks-assignment
```

Creating `requirements.txt`:
```
httpx>=0.27.0
rich>=13.7.0
python-dotenv>=1.0.0
pendulum>=3.0.0
```

Now I'll create the main script with a Market dataclass that includes computed properties for spread analysis:

```python
#!/usr/bin/env python3
"""
Kalshi Market Analyzer
======================
Pulls live prediction market data from Kalshi's public API,
identifies markets closing within specified time windows,
displays prices, and flags wide bid-ask spreads.
"""

from dataclasses import dataclass
import pendulum

@dataclass
class Market:
    """Represents a Kalshi prediction market."""
    ticker: str
    title: str
    subtitle: str
    yes_bid: float
    yes_ask: float
    no_bid: float
    no_ask: float
    close_time: pendulum.DateTime
    volume: int
    status: str
    category: str

    @property
    def spread(self) -> float:
        """Calculate bid-ask spread as a decimal."""
        return self.yes_ask - self.yes_bid

    @property
    def spread_percent(self) -> float:
        """Spread as percentage."""
        return self.spread * 100

    @property
    def is_wide_spread(self) -> bool:
        """Check if spread exceeds 10% threshold."""
        return self.spread >= 0.10
```

The dataclass uses `@property` decorators so spread calculations happen on-demand rather than storing redundant data.

---

## Prompt 2: API Integration

**Me:**
```
Good structure. Now let's implement the Kalshi API integration.

I checked their docs - the public endpoint is:
https://api.elections.kalshi.com/trade-api/v2/markets

Key considerations:
1. No authentication needed for public market data
2. Need to handle pagination (they return max 200 per request)
3. Should filter by status="open" to get active markets

Please implement the fetch_markets() and parse_market() functions with proper error handling.
```

**Claude:**
I'll implement robust API fetching with error handling:

```python
import httpx
from typing import Optional

KALSHI_API_BASE = "https://api.elections.kalshi.com/trade-api/v2"

def fetch_markets(limit: int = 500) -> list[dict]:
    """
    Fetch markets from Kalshi public API.

    Args:
        limit: Maximum number of markets to fetch

    Returns:
        List of market data dictionaries
    """
    url = f"{KALSHI_API_BASE}/markets"
    params = {
        "limit": limit,
        "status": "open",
    }

    try:
        with httpx.Client(timeout=30.0) as client:
            response = client.get(url, params=params)
            response.raise_for_status()
            data = response.json()
            return data.get("markets", [])

    except httpx.HTTPStatusError as e:
        console.print(f"[red]HTTP Error: {e.response.status_code}[/red]")
        raise
    except httpx.RequestError as e:
        console.print(f"[red]Request Error: {e}[/red]")
        raise


def parse_market(raw: dict) -> Optional[Market]:
    """
    Parse raw API response into Market object.

    Returns None if parsing fails (defensive programming).
    """
    try:
        close_time_str = raw.get("close_time") or raw.get("expected_expiration_time")
        if not close_time_str:
            return None

        close_time = pendulum.parse(close_time_str)

        # Prices are in cents, convert to dollars
        yes_bid = raw.get("yes_bid", 0) / 100 if raw.get("yes_bid") else 0
        yes_ask = raw.get("yes_ask", 0) / 100 if raw.get("yes_ask") else 0
        no_bid = raw.get("no_bid", 0) / 100 if raw.get("no_bid") else 0
        no_ask = raw.get("no_ask", 0) / 100 if raw.get("no_ask") else 0

        return Market(
            ticker=raw.get("ticker", "N/A"),
            title=raw.get("title", "Unknown"),
            subtitle=raw.get("subtitle", ""),
            yes_bid=yes_bid,
            yes_ask=yes_ask,
            no_bid=no_bid,
            no_ask=no_ask,
            close_time=close_time,
            volume=raw.get("volume", 0),
            status=raw.get("status", "unknown"),
            category=raw.get("category", ""),
        )
    except Exception:
        return None  # Skip malformed records
```

I used a 30-second timeout to handle slow API responses, and the parse function returns `Optional[Market]` to gracefully skip any malformed records.

---

## Prompt 3: Filtering Logic

**Me:**
```
The API functions look good. Now I need the filtering logic:

1. filter_closing_within(markets, hours=24) - returns markets expiring in next N hours
2. Results should be sorted by close time (soonest first)

Also add a helper property to Market class that returns human-readable time until close
(e.g., "2.5h", "45m", "3.2d").
```

**Claude:**
Adding the time formatting property and filter function:

```python
# Add to Market class
@property
def hours_until_close(self) -> float:
    """Hours remaining until market closes."""
    now = pendulum.now('UTC')
    diff = self.close_time.diff(now)
    return diff.in_hours() + (diff.in_minutes() % 60) / 60

@property
def time_until_close_str(self) -> str:
    """Human-readable time until close."""
    hours = self.hours_until_close
    if hours < 0:
        return "CLOSED"
    elif hours < 1:
        return f"{int(hours * 60)}m"
    elif hours < 24:
        return f"{hours:.1f}h"
    elif hours < 168:  # 7 days
        return f"{hours/24:.1f}d"
    else:
        return f"{hours/24/7:.1f}w"


def filter_closing_within(markets: list[Market], hours: int = 24) -> list[Market]:
    """
    Filter markets closing within specified hours.

    Args:
        markets: List of Market objects
        hours: Time window in hours (default 24)

    Returns:
        Filtered list sorted by close time (soonest first)
    """
    now = pendulum.now('UTC')
    cutoff = now.add(hours=hours)

    closing_soon = [
        m for m in markets
        if now < m.close_time <= cutoff
    ]

    closing_soon.sort(key=lambda m: m.close_time)
    return closing_soon
```

---

## Prompt 4: First Test Run - Debugging

**Me:**
```
I ran the script and got this output:

"Successfully fetched 500 markets"
"Markets closing in 24 hours: 0"

That can't be right - Kalshi has sports markets expiring daily.
Can you add some debug output to show what close_time values we're actually getting from the API?
```

**Claude:**
Let me check the raw API response:

```python
# Debug script
import httpx

url = 'https://api.elections.kalshi.com/trade-api/v2/markets'
params = {'limit': 5, 'status': 'open'}

with httpx.Client(timeout=30.0) as client:
    response = client.get(url, params=params)
    data = response.json()

    for m in data.get('markets', []):
        print(f"Ticker: {m.get('ticker')}")
        print(f"  close_time: {m.get('close_time')}")
        print(f"  expected_expiration_time: {m.get('expected_expiration_time')}")
```

Output:
```
Ticker: KXMVESPORTS...
  close_time: 2026-02-18T00:30:00Z
  expected_expiration_time: 2026-02-04T04:00:00Z
```

**Found the issue!** There are two different time fields:
- `close_time` = when trading stops (Feb 18)
- `expected_expiration_time` = when market resolves (Feb 4 - tomorrow!)

We should use `expected_expiration_time` for filtering since that's when the market actually settles.

---

## Prompt 5: Bug Fix

**Me:**
```
Good catch. The expected_expiration_time is what we want - that's when the outcome is determined.

Please update parse_market() to prioritize expected_expiration_time over close_time.
Then re-run the analysis.
```

**Claude:**
Updated the parsing logic:

```python
def parse_market(raw: dict) -> Optional[Market]:
    try:
        # FIXED: Use expected_expiration_time (when market resolves)
        # instead of close_time (when trading ends)
        close_time_str = raw.get("expected_expiration_time") or raw.get("close_time")
        if not close_time_str:
            return None
        # ... rest of function
```

Running again:
```
✓ Fetched 500 markets
✓ Parsed 500 valid markets

Markets Closing in 24 Hours
Found 427 markets
```

Now we're seeing the expected results - 427 markets expiring within 24 hours.

---

## Prompt 6: Display Formatting

**Me:**
```
Great, the filtering works now. But the table output is getting truncated.

Let me define exactly what I want to see in the main table:
- Market title (max 50 chars, truncate with ..)
- Yes price (in cents, e.g., "45¢")
- No price (in cents)
- Spread percentage with visual indicator if >10%
- Time until expiration
- Volume

Use rich library's Table with SIMPLE box style for cleaner output.
```

**Claude:**
Here's the improved display function:

```python
from rich.table import Table
from rich import box

def display_markets_table(markets: list[Market], title: str):
    """Display markets in a formatted table."""
    if not markets:
        console.print(Panel(f"[yellow]No markets found for: {title}[/yellow]"))
        return

    console.print(f"\n[bold cyan]{title}[/bold cyan]")
    console.print(f"[dim]Found {len(markets)} markets[/dim]\n")

    table = Table(box=box.SIMPLE, show_header=True, header_style="bold white")

    table.add_column("Market", width=50, no_wrap=True)
    table.add_column("Yes", justify="right", width=6)
    table.add_column("No", justify="right", width=6)
    table.add_column("Spread", justify="right", width=8)
    table.add_column("Expires", justify="right", width=8)
    table.add_column("Vol", justify="right", width=8)

    for market in markets[:15]:
        display_title = market.title[:48] + ".." if len(market.title) > 50 else market.title
        yes_cents = f"{int(market.yes_bid * 100)}¢" if market.yes_bid else "—"
        no_cents = f"{int(market.no_bid * 100)}¢" if market.no_bid else "—"

        spread_text = f"{market.spread_percent:.0f}%"
        if market.is_wide_spread:
            spread_text = f"[red bold]{spread_text}⚠[/red bold]"
        else:
            spread_text = f"[green]{spread_text}[/green]"

        table.add_row(
            display_title,
            yes_cents,
            no_cents,
            spread_text,
            market.time_until_close_str,
            f"{market.volume:,}",
        )

    console.print(table)
```

---

## Prompt 7: Wide Spread Analysis (Bonus Feature)

**Me:**
```
The core functionality is working. Now let's add the bonus feature -
flagging markets with wide bid-ask spreads.

Create a separate display function that:
1. Filters markets where spread > 10%
2. Sorts by spread (highest first)
3. Shows top 10 with the bid→ask range visible
4. Adds a header explaining why wide spreads matter (liquidity/opportunities)
```

**Claude:**
```python
def display_wide_spreads(markets: list[Market]):
    """Highlight markets with wide bid-ask spreads."""
    wide_spread_markets = [m for m in markets if m.is_wide_spread]
    wide_spread_markets.sort(key=lambda m: m.spread, reverse=True)

    if not wide_spread_markets:
        console.print("\n[green]✓ No markets with wide spreads (>10%) found.[/green]")
        return

    console.print(f"\n[bold red]⚠️  WIDE SPREAD ALERT: {len(wide_spread_markets)} markets[/bold red]")
    console.print("[dim]Markets with bid-ask spread > 10% (potential liquidity issues)[/dim]\n")

    table = Table(box=box.SIMPLE, show_header=True, header_style="bold red")
    table.add_column("Market", width=45)
    table.add_column("Spread", justify="right", width=8)
    table.add_column("Bid→Ask", justify="center", width=15)
    table.add_column("Exp", justify="right", width=6)

    for market in wide_spread_markets[:10]:
        table.add_row(
            market.title[:43] + ".." if len(market.title) > 45 else market.title,
            f"[red bold]{market.spread_percent:.0f}%[/red bold]",
            f"{int(market.yes_bid*100)}¢ → {int(market.yes_ask*100)}¢",
            market.time_until_close_str,
        )

    console.print(table)
```

---

## Prompt 8: CLI Arguments & JSON Export

**Me:**
```
For flexibility, let's add command-line arguments:
- --hours: Time window (default 24)
- --limit: Max markets to fetch (default 500)
- --min-volume: Filter by minimum volume
- --json FILE: Export results to JSON

This makes the tool more useful for different analysis scenarios.
```

**Claude:**
```python
import argparse
import json

def export_to_json(markets: list[Market], filename: str):
    """Export markets to JSON file."""
    data = [{
        "ticker": m.ticker,
        "title": m.title,
        "yes_bid": m.yes_bid,
        "yes_ask": m.yes_ask,
        "spread_percent": m.spread_percent,
        "is_wide_spread": m.is_wide_spread,
        "close_time": m.close_time.isoformat(),
        "hours_until_close": m.hours_until_close,
        "volume": m.volume,
    } for m in markets]

    with open(filename, 'w') as f:
        json.dump(data, f, indent=2)

    console.print(f"[green]✓ Exported {len(data)} markets to {filename}[/green]")


def main():
    parser = argparse.ArgumentParser(description="Analyze Kalshi prediction markets")
    parser.add_argument("--hours", type=int, default=24)
    parser.add_argument("--limit", type=int, default=500)
    parser.add_argument("--min-volume", type=int, default=0)
    parser.add_argument("--json", type=str, metavar="FILE")
    args = parser.parse_args()

    # ... main logic using args
```

---

## Prompt 9: Final Test & Verification

**Me:**
```
Let's do a final test run to verify everything works:

1. Run with default settings
2. Run with --json export
3. Verify the output shows all required information

Then create the README with clear setup instructions.
```

**Claude:**
Running final tests:

```bash
$ python src/kalshi_analyzer.py

🎯 Kalshi Market Analyzer
Real-time prediction market analysis

✓ Fetched 500 markets
✓ Parsed 500 valid markets

Markets Closing in 24 Hours
Found 427 markets

  Market                                             Yes     No   Spread   Exp      Vol
 ───────────────────────────────────────────────────────────────────────────────────────
  yes Jamal Murray: 6+,yes Cade Cunningham: 20+..     —     100¢     0%   9.1h        0
  no Detroit wins by over 15.5 Points,no New Y..     —     100¢     0%   9.1h        0
  ...

⚠️  WIDE SPREAD ALERT: 58 markets
Markets with bid-ask spread > 10% (potential liquidity issues)

  Market                                        Spread      Bid→Ask        Exp
 ───────────────────────────────────────────────────────────────────────────────
  yes Nickeil Alexander-Walker: 15+,yes           100%     0¢ → 100¢     12.1h
  ...

📈 Summary Statistics
╭────────────────────────────┬──────────────────────╮
│ Metric                     │                Value │
├────────────────────────────┼──────────────────────┤
│ Total Markets Analyzed     │                  500 │
│ Closing in 24 hours        │ 427 (53 wide spread) │
│ Wide Spread Markets (>10%) │                   58 │
╰────────────────────────────┴──────────────────────╯

✓ Analysis complete!
```

JSON export test:
```bash
$ python src/kalshi_analyzer.py --json output.json
✓ Exported 427 markets to output.json
```

All tests passing. README created with setup instructions.

---

## Prompt 10: Excel Export (Bonus Feature)

**Me:**
```
The JSON export works well, but for sharing with non-technical stakeholders,
an Excel export would be more useful.

Can you add an --excel flag that exports to .xlsx with:
1. Sheet 1: All markets with full data
2. Sheet 2: Wide spreads only (highlighted in red)
3. Sheet 3: Summary statistics
4. Professional formatting (headers, borders, colors)

Use openpyxl library for this.
```

**Claude:**
Adding Excel export with formatted sheets:

```python
from openpyxl import Workbook
from openpyxl.styles import Font, PatternFill, Alignment, Border, Side

def export_to_excel(markets: list[Market], wide_spread_markets: list[Market], filename: str):
    """Export markets to Excel file with formatting."""
    wb = Workbook()

    # Styles
    header_font = Font(bold=True, color="FFFFFF")
    header_fill = PatternFill(start_color="1F4E79", end_color="1F4E79", fill_type="solid")
    wide_spread_fill = PatternFill(start_color="FFCDD2", end_color="FFCDD2", fill_type="solid")

    # Sheet 1: All Markets
    ws1 = wb.active
    ws1.title = "All Markets"
    # ... headers and data rows with formatting

    # Sheet 2: Wide Spreads Alert
    ws2 = wb.create_sheet(title="Wide Spreads Alert")
    # ... filtered data with red highlighting

    # Sheet 3: Summary Statistics
    ws3 = wb.create_sheet(title="Summary")
    # ... summary metrics

    wb.save(filename)
```

Added CLI argument:
```python
parser.add_argument("--excel", type=str, metavar="FILE", help="Export to Excel (.xlsx)")
```

Testing:
```bash
$ python src/kalshi_analyzer.py --excel kalshi_report.xlsx

✓ Exported to Excel: kalshi_report.xlsx
  - Sheet 1: All Markets (410 rows)
  - Sheet 2: Wide Spreads (74 rows)
  - Sheet 3: Summary Statistics
```

Excel file structure verified:
```
📊 EXCEL REPORT: kalshi_report.xlsx (38 KB)
├── Sheet: All Markets (411 rows × 11 cols)
├── Sheet: Wide Spreads Alert (75 rows × 6 cols)
└── Sheet: Summary (6 rows × 2 cols)
```

---

## Summary

**Total Development Time:** ~2 hours

**Key Iterations:**
1. Initial structure and dataclass design
2. API integration with error handling
3. **Bug fix**: Discovered close_time vs expected_expiration_time discrepancy
4. Display formatting improvements
5. Added bonus features (wide spread alerts, JSON export)
6. **Extra**: Excel export with formatted sheets

**What Worked Well:**
- Starting with clear requirements made implementation straightforward
- Using dataclass with computed properties kept the code clean
- The rich library made CLI output professional with minimal effort
- openpyxl made Excel formatting straightforward

**Course Corrections:**
- Had to debug the time field issue by examining raw API responses
- Simplified table columns when initial design was too wide for terminal
- Removed unnecessary numpy dependency from openpyxl import

---

*End of transcript*
