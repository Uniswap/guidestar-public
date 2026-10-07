# Guidestar

Guidestar is the Uniswap Auto Router. It finds the route for a swap quote across Uniswap v2, v3, and v4 pools, v4 hooks, and registered external venues.

This repository publishes two things. The first is a description of how the router chooses a route. The second is the configuration that the router uses in production, with its live values.

## Repository contents

- `config/<chain_id>.json` is the router configuration for one chain. The repository has a file for each of these chains:
  - `1.json`: Ethereum
  - `8453.json`: Base
  - `130.json`: Unichain
  - `4663.json`: Robinhood Chain
- `config/permissioned-hooks.json` is the permissioned-hooks registry for all chains.

An automated job copies these files from the router's production source each time that source changes. Do not edit them in this repository.

The section "Router Configuration" below names each variable as it appears in the router code. The configuration files use these keys:

| Variable | Key in `config/<chain_id>.json` |
|---|---|
| `UniswapV2Factory`, `UniswapV3Factory`, `PoolManager` | `factories` |
| `ADHOC_HOOK_QUOTES_ENABLED` | `adhoc_hook_quotes_enabled` |
| `UNKNOWN_HOOK_ADMISSION_ENABLED` | `unknown_hook_admission_enabled` |
| `GUIDESTAR_AGG_HOOK_DISCOVERY_SEC` | `aggregator_hook_discovery_sec` |
| `GUIDESTAR_AGG_HOOK_BOUNDS_REFRESH_SEC` | `aggregator_hook_bounds_refresh_sec` |

The other variables use the same name as the key. The permissioned-hooks registry is in `config/permissioned-hooks.json`.

## How the Auto Router chooses a route

### What the Router Does

When you ask for a swap quote, the router ("Guidestar") searches for a path to carry out your trade that is expected to leave you with the most of the token you are buying after network fees (the "most efficient path"). Or, if you fixed the amount you want to receive, the path expected to cost you the least. It computes this fresh for every request.

There are four types of inputs which go into the routing decision:

1. **Your request.** The token pair, the amount, the trade direction, your slippage tolerance and deadline, and which liquidity sources (protocol versions and external venues) you allow. You set all of these, and you can change them.
2. **Public blockchain state.** The prices, reserves, and liquidity of eligible pools as of the latest block, and the network's current gas price. Anyone can read the same data from the chain.
3. **The published configuration** — the file that accompanies this disclosure. It contains the variables that define the universe of tokens and pools the router selects from: the official Uniswap contract addresses whose pools count as canonical, the contract addresses of the recognized external protocols the router can also draw on, the widely-traded "common tokens" the router may route through, fallback route orderings, the blocked-pool list, and the registered and vetted hook-venue lists. (A full configuration explanation is provided at the end of this document.)
4. **Token-safety screens.** A continuously updated list that keep problematic tokens out of the middle of a route: tokens blocked for compliance, tokens flagged as spam, and tokens that take a fee on transfer.

### How the Pool Universe is Chosen

The router finds a path through a network of pools. The network of pools which the router can use is continuously updated based on the public blockchain state. But, not every pool is eligible for the router to use. A pool passes a set of eligibility checks before it is added to the router’s search space:

- It was created by the official Uniswap v2, v3, or v4 contracts listed in the published configuration and is unhooked or hooked with standard math (e.g. not a custom accounting or dynamic fee hook).
- It was created by the official Uniswap v4 contracts listed in the published configuration, is hooked with a custom accounting or dynamic fee hook, and is included in the list of allowed hooks provided in the configuration.
  - This includes external AMMs wrapped through “aggregator hooks” which are discovered on-chain from a registered source (see the configuration). These pools are priced using that AMM’s native math or by quoting an RPC. If priced using an RPC, the pool must be capable of returning successful quotes at multiple trade sizes. A quote is successful if the venue responds without error for the requested trade.
- It has usable liquidity right now; empty or broken pools are skipped. Pools with sparse or inconsistent liquidity may be capped in the trade portion size permitted to route through them or may be skipped.
- It has shown trading activity within the past 60 days. Pools with 0 trading in the past 60 days are not eligible for routing.
- Pools that cause simulation or routing failures are added to the published blocked-pool list, with tokens cross-checked against the [Unsupported Token List](https://unsupportedtokens.uniswap.org/).

### How the Route is Chosen

The router treats tokens as points on a map and pools as roads between them (collectively called a “graph”), and searches that network using a [Bellman–Ford-style graph algorithm](https://en.wikipedia.org/wiki/Bellman%E2%80%93Ford_algorithm) — a standard, textbook method for finding the shortest path through a network, adapted here to maximize the amount you receive rather than minimize a distance. From there the router works through four stages: narrow the graph nodes to your trade, search the graph for the most efficient path, decide whether to split the order, and apply network costs to pick a single winner.

**Step 1 — Vetting the pools in play for this trade.** The pool universe above is chain-wide; a single trade only needs the portion of it that can touch your two tokens. At request time the router keeps the pools that connect your tokens directly, the pools that link them through the common tokens, and the external venues and hook pools (depending on your liquidity-source selections in your request). The router determines which intermediate tokens beyond the common list may appear mid-route by trial-quoting each candidate with fixed-size trades of no more than 10 ETH in value, and only candidates competitive (within at most 5%) with the most efficient alternative already found are admitted to the graph. Each surviving pool is re-checked against the latest block. Both of its tokens must be priceable, have required recent activity, and pass the blocked-pool and token-safety screens. Pools gated by permissioned (allowlist) hooks stay on the graph only when the gated token is one end of your trade, never as an in-between hop. Pools whose live on-chain liquidity is too small for the size being considered are left off the graph.

As an additional check, when "Adhoc Hooks" are enabled (see configurations below) the router checks if there is any hooked pool meeting a minimum depth threshold that is not lower than the liquidity of the tenth-deepest unhooked pool for the relevant input or output token and which has not yet been considered. (This depth check may be bypassed if the hook is on the manually vetted activation list published in the configs.) If so, that hooked pool is checked against the latest block to ensure it’s priceable and added to the pools which the routing algorithm will search.

**Step 2 — Searching for the most efficient paths.** On this trade-specific graph, the Bellman–Ford-style search starts at the token you are selling and repeatedly extends routes found so far, one pool at a time. On each pass the search keeps the most efficient path; a path survives only by strictly improving on the previous quantity of output token. Every evaluation is computed from the pools' current on-chain state and accounts for how your own trade would move each pool's price as it executes. The pre-set fallback route orderings from the published configuration are scored alongside the router’s solution, and the most efficient of all candidates moves forward.

**Step 2a — Deciding whether to split the order.** The router does not commit the whole amount to one path up front: it routes the trade in increments, and as earlier increments impact a given pool's prices, the next increment may find that a different pool is now more favorable. Therefore a large order typically spreads across several pools or paths because exactly when the leading pool's price deteriorates, an alternative wins. Splitting is never assumed to be better: more-divided and less-divided versions of the trade are compared against each other, and a split survives only if it still delivers a better overall result after the added network cost of touching more pools.

**Step 3 — Charging network costs and picking the winner.** Finally, every candidate is charged its estimated network (gas) cost, priced at the network's current gas rate and converted into the token you are receiving using market prices observed in the same pool network — so tokens received and gas spent are compared in the same units. For exact-input trades the router keeps the candidate that delivers the most after that charge. For exact-output trades, the one that requires the least including it. The single winning route is then simulated against the chain to confirm it behaves as expected and to refine the returned quote and gas estimate.

### What the Router Outputs

The chosen route (which includes each pool and the amount sent into it), the quoted amount (e.g. the estimated output), and a gas estimate priced at the network's current rate are returned by the router. If no route can carry the full amount, or no path can be found to complete the route, the router returns no route.

### Router Configuration

The published configuration file contains, per chain, exactly the variables below. Each is listed here by its exact name in the code, with what it controls in the process described above. The accompanying configuration file presents these variables with their live values.

- `UniswapV2Factory`, `UniswapV3Factory`, `PoolManager` — the official Uniswap contract addresses whose pools count as canonical members of the pool universe.
- `aggregator_hook_sources` — per external protocol (`protocol`), the one on-chain contract the router scans to discover and admit that protocol's routable venue pools: the Uniswap-deployed aggregator-hook wrapper (`hook`) or the protocol's own pool registry (`factory`), optionally capped by `maxRecords`.
- `common_tokens` — the widely-traded tokens every route may pass through as intermediate hops (the "short detours" of Step 2) and that serve as price references.
- `fallback_orders` — the pre-set token orderings scored alongside the graph search's own answer in Step 2; the most efficient of all candidates is the route returned.
- `blocked_pool_ids` — pools excluded from routing and from price references outright, regardless of every other rule.
- `hooks` — the standing registry of individually registered v4 hook pools, one entry per pool (unlike the hook-level lists below); each entry identifies its pool by `poolKey` (`primaryToken`, `secondaryToken`, `fee`, `tickSpacing`, `hooks` address) and `poolId`, and `customQuote` records whether the pool is priced from trial quotes instead of standard pool math.
- Permissioned-hooks registry (`chains` → `hookAddress`, `adapters` with `address`, `decimals`, `underlyingToken`) — the allowlist of permission-gated hooks and their gated tokens, which Step 1 admits only when the gated token is an endpoint of your trade.
- `adhoc_hooks` — the manually vetted hook addresses (per entry: `address`, `bypassDepthGate`) whose pools may be trial-quoted into a request even where the default ad-hoc safety gates would exclude them; hook-level and per-request — one entry covers every pool on that hook, and nothing joins the standing registry.
- `ADHOC_HOOK_QUOTES_ENABLED` — the on/off switch for an optional Step 1 feature that trial-quotes not-yet-registered hook pools into a single request's graph.
- `UNKNOWN_HOOK_ADMISSION_ENABLED` — the on/off switch that admits unregistered hook pools to the standing pool universe when the hook stays out of the swap path and standard math can price the pool.
- `GUIDESTAR_AGG_HOOK_DISCOVERY_SEC` — how often, in seconds, the router re-scans the registered sources for newly created venue pools.
- `GUIDESTAR_AGG_HOOK_BOUNDS_REFRESH_SEC` — how often, in seconds, registered venues' quoting set-up is refreshed and every source is fully re-enumerated.

### Additional Pre-Set Routing Parameters

In addition to the published configuration above, the router applies a small set of pre-decided parameters inside its search. Each is fixed in code, applied uniformly to every user and every request, and is not user-configurable.

- **Order increment size** — the swap is routed in equal increments, each a fixed fraction of the order greater than 0.1%.
- **Leg-removal test threshold** — route legs carrying no more than 20% of the flow are tested for removal: does their network cost outweigh their price improvement contribution.
- **Quote tolerances** — a pool's recent quote may be reused instead of recomputed when it is estimated to have not changed (by less than 10%) and a quote that would reverse flow already planned through a pool is discounted by 0.001% of its value or 100 of the token's smallest indivisible units, whichever is greater.
- **Gas accounting adjustments** — each hop's measured gas, and the cost of converting between native and wrapped ETH, are adjusted by fixed allowances of less than 50,000 gas each.
- **Gas-comparison price probes** — gas costs and leg sizes are converted to comparable units for measurement. When necessary, price probes, each no more than 0.1 ETH or 100 USD in value, are used to estimate the comparable unit.

## License

MIT License. See [LICENSE](LICENSE).
