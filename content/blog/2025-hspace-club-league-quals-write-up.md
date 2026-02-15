+++
title = "2025 HSPACE CLUB LEAGUE Quals Write Up"
date = 2026-02-15
description = "Write-up scaffold for the 2025 HSPACE CLUB LEAGUE Quals challenges"
draft = false

[taxonomies]
tags = ["CTF"]
+++

## TL;DR
- Perpdex challenge serving `ETH/USD`, `BTC/USD`, `SOL/USD`, and `HYPE/USD` pairs; `Setup.sol` seeds the solver with the initial bankroll needed to open and hedge positions.
- Solve condition from the challenge statement (must use only the provided bankroll and hit the gate below).

## Challenges
- babyevm (web3, 900 points): summary and key bug.

## Setup Contract
- `Setup.sol` deploys `PriceOracle`, `USDH`, `HspacePerpDEX`, approves and deposits `500_000 ether - 100 ether` into the DEX, and gives the solver the remaining `USDH` via `claim()`.

```solidity
pragma solidity ^0.8.25;
import "./HspacePerpDEX.sol";
import "./PriceOracle.sol";
import "./USDH.sol";

contract Setup {
    HspacePerpDEX public dex;
    PriceOracle public oracle;
    USDH public usdh;
    mapping(address => uint256) public donations;
    bool public isSolved;
   
    uint256 public initialBalance = 500_000 ether - 100 ether;
    constructor() {
        oracle = new PriceOracle();
        usdh = new USDH();
        dex = new HspacePerpDEX(address(oracle), address(usdh));
        usdh.approve(address(dex), initialBalance);
        dex.deposit(initialBalance);
    }

    function claim() public {
        // 100 USDC 제공
        usdh.transfer(msg.sender, usdh.balanceOf(address(this)));
    }

    function solve() public {
        bytes32[] memory markets = new bytes32[](4);
        markets[0] = dex.ETH_USD_MARKET();
        markets[1] = dex.BTC_USD_MARKET();
        markets[2] = dex.SOL_USD_MARKET();
        markets[3] = dex.HYPE_USD_MARKET();

        // 위 중 하나라도 아래 조건을 모두 만족해야 함
        for (uint256 i = 0; i < 4; i++) {
            (uint256 totalTrades,,uint256 winRate,,,int256 roi) = dex.getTraderStats(msg.sender, markets[i]);
            // over 20 trades, 80% win rate, 100000000000000% ROI
            if (totalTrades < 20 || winRate < 8000 || roi < 10_000_000_000_000_000) {
                continue;
            }
            isSolved = true;
            break;
        }
        return;
    }
}
```

Oracle price walk (core loop):
```python
def update_simulated_prices(self):
    # random drift between -0.1% and +0.1%, repeated 5 times
    for _ in range(5):
        for token in self.prices:
            change_percent = random.uniform(-0.1, 0.1) / 100.0
            self.prices[token] *= (1 + change_percent)
```
The listener runs this every 5 seconds and pushes `ETH/USD`, `BTC/USD`, `SOL/USD`, and `HYPE/USD` via `updatePrices`, so the feed is a gentle random walk unless you seize oracle control.

## Exploitation Notes
- ROI hack: `getTraderStats` divides PnL by `deposit_sum`; direct `receive` funding boosts balance without increasing `deposit_sum`, so ROI explodes with tiny reported capital.
- Win-rate/ROI formulas: `winRate = (winningTrades * BASIS_POINTS) / totalTrades` and `roi = (totalPnL * BASIS_POINTS) / deposit_sum[trader]` keep wins-only counting and make ROI skyrocket when `deposit_sum` is tiny.

```solidity
function getTraderStats(address trader, bytes32 marketId) external view returns (
    uint256 totalTrades,
    uint256 winningTrades,
    uint256 winRate,
    int256 totalPnL,
    uint256 totalVolume,
    int256 roi
) {
    // ...
    
    if (totalTrades > 0) {
        winRate = (winningTrades * BASIS_POINTS) / totalTrades; // 이긴 거래 횟수 * 10000 / 총 거래 횟수
    } else {
        winRate = 0;
    }
    if (balances[trader] > 0) {
        roi = (totalPnL * int256(BASIS_POINTS)) / int256(deposit_sum[trader]);
    } else {
        roi = 0;
    }
}

receive() external payable {
    balances[msg.sender] += msg.value;           // no deposit_sum bump
    totalLiquidity += msg.value;
}

function deposit(uint256 amount) external {
    // deposit_sum only increases here
    balances[msg.sender] += amount;
    deposit_sum[msg.sender] += amount;
}
```
- Win-rate shaping: `closePosition` increments `winningTrades` only when `pnl > 0`, so only profitable closes lift win rate. `liquidate` can be called on your own underwater positions to exit without touching `winningTrades`. By polling `getPosition` for unrealized PnL, you can liquidate losers and close winners, driving `winRate` to ≥80% while pairing with the ROI boost from the `deposit_sum` gap.

```solidity
function closePosition(bytes32 marketId) public whenNotPaused {
    // ...
    
    stats.totalTrades += 1;
    if (pnl > 0) {
        stats.winningTrades += 1;
    }
    
    // ...
    
    delete positions[msg.sender][marketId];
    
    // ...
}

function liquidate(address trader, bytes32 marketId) external whenNotPaused {
    Position storage pos = positions[trader][marketId];
    require(pos.size > 0, "No position to liquidate");
    
    // ...

    delete positions[trader][marketId];
    
    // ...
}

function getPosition(address trader, bytes32 marketId) external view returns (
    bool isLong,
    uint256 size,
    uint256 collateral,
    uint256 entryPrice,
    uint256 leverage,
    int256 unrealizedPnL
) {
    Position storage pos = positions[trader][marketId];
    return (
        pos.isLong,
        pos.size,
        pos.collateral,
        pos.entryPrice,
        pos.leverage,
        calculatePnL(trader, marketId)
    );
}
```

### Payload scripts (trimmed to core actions)
- setup.s.sol: `claim()` to grab USDH, `approve` + `deposit(1)` to seed `deposit_sum` minimally, then `dex.call{value: 0.95 ether}("")` to inflate `balances` without touching `deposit_sum`.
- open.s.sol: fetch `getLatestPrice` for entry, (optionally) open a tiny position, and use `getTraderStats` to monitor `totalTrades/winRate/ROI` progress.
- close.s.sol: read `getPosition` for PnL; if `pnl > 0` then `closeETHPosition()` (win++), else `liquidate(eoa, market)` to reset a loser without harming win count; loop and re-check `getTraderStats`.

## Notes
- Official solve hinged on frontrunning oracle price updates in mempool and trading with higher gas at settle time. My approach leaned on ROI/win-rate manipulation via `deposit_sum` gap and self-liquidations—an unintended path.
