# Monad Dex

| <img src="https://files.catbox.moe/ujzf30.gif" alt="USDC" width="36" height="36"> | USDC | USD Coin | `0x754704Bc059F8C67012fEd69BC8A327a5aafb603` |
| <img src="https://files.catbox.moe/lk7mrk.png" alt="BTCF" width="36" height="36"> | BTCF | BitcoinFlash | `0x7d7E0112d2763c98238aF7cebEAe33d53F3F76DD` |
| <img src="https://files.catbox.moe/k0ibzr.png" alt="MDX"  width="36" height="36"> | MDX  | MDX COIN | `0x66238b63532509A9c9C2440DE341fd2335794a07` |

This repository contains a small Uniswap-v2 style DEX prototype deployed on the Monad network (chainId 143). It includes token contracts, a factory/pair implementation, a router, a MasterChef, an ActivityTracker, and convenience scripts to deploy, mint BFL, and add liquidity.

This README documents:
- Project overview
- Files and contracts
- Public/important functions (what they do and how to call)
- Scripts and usage (deploy, verify, mint, add liquidity)
- Verification steps
- Quick checks and troubleshooting
- Security notes

---

## Project overview

This prototype demonstrates:
- An ERC-20 token (MDX) for the DEX governance/rewards
- A BitcoinFlash token (BFL) used as a token in a pair with USDC
- UniswapV2-style `UniswapV2Factory` + `UniswapV2Pair` pair contracts
- `UniswapV2Router02` for convenient liquidity and swap operations
- `MasterChef` for staking/rewards (MDX)
- `ActivityTracker` (prototype usage)
- Scripts (ethers v6) for minting BFL and adding liquidity through the pair contract
- Foundry deploy script for on-chain deployment and verification helper commands

Deployed addresses (from your recent deploy)
- MDXToken: 0x66238b63532509A9c9C2440DE341fd2335794a07
- Factory: 0x759774EbC4d5C5c83a255A14A25464cAD9dc4B3F
- Router: 0x7139332aa7C461bfC6463586D0fbf5A7cdEf5324
- Pair MDX-USDC: 0x72eEC70b6058bAc9c726cAA08cc02406aC1d7E23
- Pair BitcoinFlash-USDC: 0x57489A99059A41464fE9322864E3B241D96c5d62
- MasterChef: 0x8Dd025006D20aE484E9ac03D044fAaBA952D321F
- ActivityTracker: 0xb007C8E63FdF73989bb30B8866C5b0863fa9853a

---

## Repo files (important)
- `script/Deploy.s.sol` — Foundry deploy script (deploys MDX, Factory, Router, pairs, MasterChef, ActivityTracker)
- `src/MDXToken.sol` — MDX token (ERC20 + ownership)
- `src/UniswapV2Factory.sol` — Factory contract (createPair, getPair)
- `src/UniswapV2Router02.sol` — Router contract (addLiquidity, swapExactTokensForTokens, etc.)
- `src/UniswapV2Pair.sol` — Pair contract (mint, getReserves, LP accounting)
- `src/MasterChef.sol` — Staking / rewards contract
- `src/ActivityTracker.sol` — Activity tracker helper contract
- `scripts/mintBFL.js` — Node script (ethers v6) to mint BFL (owner-only)
- `scripts/addLiquidity.js` — Node script (ethers v6) to seed a pair: transfers tokens to pair and calls `pair.mint(recipient)`
- `README.md` — this file

---

## Environment variables (.env)

Create a `.env` (project root) with these keys. Do NOT commit your private key to git.

Example:
```
RPC_URL="https://monad-mainnet.g.alchemy.com/v2/<YOUR_ALCHEMY_KEY>"
PRIVATE_KEY="0xYOUR_PRIVATE_KEY"          # signer used for deploy and scripts (keep secret)
FACTORY_ADDRESS="0x759774EbC4d5C5c83a255A14A25464cAD9dc4B3F"
ROUTER_ADDRESS="0x7139332aa7C461bfC6463586D0fbf5A7cdEf5324"
USDC_ADDRESS="0x754704Bc059F8C67012fEd69BC8A327a5aafb603"
BITCOIN_FLASH_ADDRESS="0x7d7E0112d2763c98238aF7cebEAe33d53F3F76DD"
RECIPIENT="0x592B35c8917eD36c39Ef73D0F5e92B0173560b2e"   # optional; defaults to signer
AMOUNT_USDC="1"
AMOUNT_BFL="0.0001"
```

Note: if you earlier exposed a private key in logs or chat, treat it as compromised and rotate it immediately.

---

## Contracts — key functions and usage

This section lists the most important externally-visible functions used by the scripts and deployment. It is a summary (not the full contract source). Use these functions via ethers/cast/curl JSON-RPC calls.

### MDXToken (src/MDXToken.sol)
Typical ERC20 + Ownable-like interface:
- constructor(string name, string symbol)
- transfer(address to, uint256 amount)
- approve(address spender, uint256 amount)
- allowance(address owner, address spender)
- balanceOf(address owner) -> uint256
- totalSupply() -> uint256
- transferOwnership(address newOwner)

Usage: deployed during `Deploy.s.sol`. Ownership is transferred to MasterChef.

### UniswapV2Factory (src/UniswapV2Factory.sol)
Important functions:
- constructor(address feeToSetter) — set feeToSetter (deployer EOA used in this repo)
- createPair(address tokenA, address tokenB) -> address — creates pair
- getPair(address tokenA, address tokenB) -> address — returns pair address (ZeroAddress if not exists)

Usage: `Deploy.s.sol` creates two pairs (MDX/USDC and BitcoinFlash/USDC).

### UniswapV2Router02 (src/UniswapV2Router02.sol)
Important functions:
- constructor(address factory, address WETH) — set factory and WETH addresses
- addLiquidity(address tokenA, address tokenB, uint amountADesired, uint amountBDesired, uint amountAMin, uint amountBMin, address to, uint deadline) -> (uint amountA, uint amountB, uint liquidity) — add liquidity to a pair
- swapExactTokensForTokens(uint amountIn, uint amountOutMin, address[] path, address to, uint deadline) -> uint[] amounts — swap exact tokens for tokens
- getAmountsOut(uint amountIn, address[] path) -> uint[] amounts — get output amounts for a swap

Usage: The router provides a convenient interface for adding liquidity and swapping tokens. It handles token transfers and pair creation automatically.

### UniswapV2Pair (src/UniswapV2Pair.sol)
Important functions:
- mint(address to) -> uint256 — mints LP tokens to `to` after tokens were transferred to the pair
- getReserves() -> (uint112 reserve0, uint112 reserve1) — read reserves
- balanceOf(address owner) -> uint256 — LP balance (ERC20)
- transfer/approve/etc (ERC20 functions)

Usage: `scripts/addLiquidity.js` transfers tokens to pair contract and calls `mint(recipient)`.

### BitcoinFlash token (BFL)
Important functions used by scripts:
- mint(address to, uint256 amount) — owner-only mint
- decimals() -> uint8
- owner() -> address
- balanceOf(address owner) -> uint256

Usage: `scripts/mintBFL.js` mints BFL to a recipient. The script validates RECIPIENT and falls back to signer if invalid.

### MasterChef (src/MasterChef.sol)
Important (deployer actions):
- constructor(address mdx) — deploy and then MDX ownership is transferred to MasterChef by the deploy script
- (other staking/reward functions typical to MasterChef — deposit, withdraw, update, etc.)

### ActivityTracker (src/ActivityTracker.sol)
- constructor(address nft, uint256 threshold)

---

## Scripts — how to run

All scripts use ethers v6 (JsonRpcProvider and Wallet). Make sure you have Node (>=16) installed and `npm install` was run if you depend on additional packages.

1) Deploy with Foundry (broadcast)
- Export env in shell:
```bash
export RPC_URL="https://monad-mainnet.g.alchemy.com/v2/<KEY>"
export PRIVATE_KEY="0xYOUR_PRIVATE_KEY"
export USDC_ADDRESS="0x754704Bc059F8C67012fEd69BC8A327a5aafb603"
export BITCOIN_FLASH_ADDRESS="0x7d7E0112d2763c98238aF7cebEAe33d53F3F76DD"
export ROUTER_ADDRESS="0x7139332aa7C461bfC6463586D0fbf5A7cdEf5324"
export NFT_ADDRESS="0x1eE15cdB9D241fe7e182259F5513ff89f9165b6F"
```
- Run Foundry deploy:
```bash
forge script script/Deploy.s.sol:Deploy --rpc-url "$RPC_URL" --private-key "$PRIVATE_KEY" --broadcast 2>&1 | tee deploy.log
```
- Capture addresses printed in the deploy log.

2) Mint BFL
- Script: `scripts/mintBFL.js`
- Validates the RECIPIENT env var (falls back to signer).
- Usage:
```bash
node scripts/mintBFL.js
# or override amounts
AMOUNT_BFL="0.0001" RECIPIENT="0x..." node scripts/mintBFL.js
```

Important behavior:
- If RECIPIENT is a placeholder or not a valid address, the script will use the signer address to prevent ENS lookup errors on networks that do not support ENS (like Monad RPC used here).

3) Add liquidity (seed pair)
- Script: `scripts/addLiquidity.js`
- Behavior:
  - Validates FACTORY_ADDRESS / USDC_ADDRESS / BITCOIN_FLASH_ADDRESS
  - Finds pair via factory.getPair(USDC, BFL)
  - Transfers the requested USDC and BFL from signer to the pair
  - Calls `pair.mint(recipient)`
- Usage:
```bash
# using .env values
node scripts/addLiquidity.js

# or override amounts inline
AMOUNT_USDC="1" AMOUNT_BFL="0.0001" node scripts/addLiquidity.js
```

Notes:
- Scripts use checks (ethers.isAddress / ethers.getAddress) to avoid ENS resolution attempts and to normalize addresses. This avoids the "network does not support ENS" error.

---

## Verification (Sourcify) — steps you ran

Commands used to create constructor arg files (examples with `cast`):
```bash
cast abi-encode "constructor(string,string)" "MonadDex Token" "MDX" > mdx_ctor.hex
cast abi-encode "constructor(address,address)" 0x754704... 0x7d7E0... > pair_ctor.hex
cast abi-encode "constructor(address,uint256)" 0x1eE15c... 1000000 > tracker_ctor.hex
cast abi-encode "constructor(address)" 0x592B35... > factory_ctor.hex
```

Submitted Sourcify verification with `forge verify-contract`:
```bash
forge verify-contract --rpc-url https://rpc.monad.xyz --verifier sourcify --verifier-url 'https://sourcify-api-monad.blockvision.org/' --chain-id 143 --constructor-args mdx_ctor.hex 0x6623... src/MDXToken.sol:MDXToken
# ... similarly for pair, factory, router, activity tracker
```

You can check job status via the Sourcify job URLs returned by `forge` (the commands printed verification job IDs and URLs).

---

## Quick on-chain checks (using `cast` from Foundry)

Check contract code:
```bash
cast rpc eth_getCode 0x759774EbC4d5C5c83a255A14A25464cAD9dc4B3F --rpc-url "$RPC_URL"
# "0x" -> not deployed; non-empty -> deployed
```

Check tx receipt:
```bash
cast tx <TX_HASH> --rpc-url "$RPC_URL"
```

Call contract read functions (example `getReserves()`):
```bash
cast call --rpc-url "$RPC_URL" --to 0x57489A99059A41464fE9322864E3B241D96c5d62 --data "$(cast calldata "getReserves()")"
```

Get LP token balance:
```bash
cast call --rpc-url "$RPC_URL" --to 0x57489A99059A41464fE9322864E3B241D96c5d62 --data "$(cast calldata "balanceOf(address)" 0x592B35...)" | xargs -I{} cast --to-hex {} && echo
```

(If you want, I can provide pretty-print helper scripts that decode the results and show decimals.)

---

## Troubleshooting & common errors

- "network does not support ENS" — occurs when ethers tries to resolve a name because a provided address string is invalid. Fix by passing a valid checksum address or using ethers.getAddress / ethers.isAddress for validation. Scripts already validate addresses to avoid this problem.
- "odd number of digits" or "invalid address" — make sure you paste a full 0x prefixed 40-hex-digit address.
- Foundry "Usage of `address(this)` detected in script contract" — use `msg.sender` (EOA) as the factory constructor arg in script after `vm.startBroadcast()`.
- 401 from RPC provider — your RPC_URL (Alchemy/Infura) must be a valid authenticated endpoint.

---

## Security recommendations

- Do NOT commit PRIVATE_KEY or secrets to git. Use environment variables in your shell or a secrets manager.
- If you believe a private key was exposed, immediately:
  - Move funds to a new wallet (new private key)
  - Revoke token approvals from the compromised wallet if appropriate
- Consider using a hardware wallet or multisig for production deployments.

---

## Next helpful scripts I can add (offer)

If you want, I can add the following helper files to the repo:
- `scripts/checkLP.js` — prints pair reserves and LP balance in human readable units
- `scripts/verifyStatuses.sh` — polls Sourcify verification job statuses and prints result
- `scripts/rotate-key.md` — step-by-step instructions to create a new key, transfer funds, and secure the repo

Tell me which helper(s) you want added and I will create them.

---

If you'd like any changes to this README (different format, more contract ABI details, or to include actual code snippets from each contract), tell me what to include and I'll update it.
