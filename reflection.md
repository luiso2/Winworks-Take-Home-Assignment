# Reflection

## My Approach

I chose the Kalshi option because prediction markets offer clean, structured data with a well-documented public API. My strategy was to break the problem into clear phases: data fetching, parsing, filtering, and display. Rather than asking the AI for a complete solution upfront, I provided specific requirements for each component—this gave me more control over the architecture and made debugging easier.

I started by defining the data model (Market dataclass) with computed properties for spread calculations. This decision paid off because it kept the business logic encapsulated and made the filtering/display code much cleaner. I specified my preferred tech stack (httpx, rich, pendulum) in the initial prompt to ensure consistency.

## Where I Course-Corrected

The main debugging moment came when the initial run showed zero markets closing in 24 hours—clearly wrong for a platform with daily sports markets. Instead of asking the AI to "fix it," I requested debug output to examine the raw API response. This revealed that Kalshi uses two different time fields: `close_time` (when trading ends) and `expected_expiration_time` (when the market resolves). The fix was straightforward once I understood the data model.

I also had to iterate on the display formatting. My first table design had too many columns and got truncated in standard terminal widths. I simplified to the essential fields and switched price display from dollars to cents, which is more intuitive for prediction market users where prices represent probabilities.

## What I'd Do Differently

With more time, I would add category filtering—currently sports parlays dominate the output, and political/economic markets might be more interesting for analysis. I'd also implement historical tracking to identify markets where spreads are widening over time, which could signal changing sentiment.

The AI tooling was most valuable for boilerplate (rich table syntax, argparse setup) and for quickly iterating on display formatting. The key was being specific about requirements and treating the AI as a skilled implementer rather than an architect—I made the design decisions, then delegated the implementation details.
