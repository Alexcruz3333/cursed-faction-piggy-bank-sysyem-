
---

# PiggyBank NFT — What it does

* **Mintable ERC-721**: each NFT has its own vault.
* **Deposits**: send ETH (and optional ERC-20s) “into” a token’s vault.
* **Ownership = control**: only the **current NFT owner** can withdraw.
* **Giftable**: anyone can top up your piggy-bank NFT (great for tips, gifts, prize pools).
* **Balances follow the token**: sell/transfer the NFT and the vault balance goes with it.
* **Optional goal + lock**: set a savings goal or a time lock if you want (toggleable).

---

## Solidity (single contract)

Paste this into `PiggyBankNFT.sol`. Uses OpenZeppelin.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * PiggyBank NFT
 * - Each ERC721 tokenId has a native-ETH vault and optional ERC20 vaults.
 * - Anyone can deposit; only current owner can withdraw.
 * - Balances follow the token on transfer.
 */

import "@openzeppelin/contracts/token/ERC721/ERC721.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import "@openzeppelin/contracts/token/ERC20/IERC20.sol";

contract PiggyBankNFT is ERC721, Ownable, ReentrancyGuard {
    using SafeERC20 for IERC20;

    uint256 public nextId = 1;
    string private _baseTokenURI;

    // ETH balances by tokenId
    mapping(uint256 => uint256) public ethBalance;

    // ERC20 balances by tokenId => token => amount
    mapping(uint256 => mapping(address => uint256)) public erc20Balance;

    // Optional savings goal and unlock (unix timestamp)
    mapping(uint256 => uint256) public savingsGoal;
    mapping<uint256 => uint256) public unlockTime;

    event DepositedETH(uint256 indexed tokenId, address indexed from, uint256 amount);
    event WithdrawnETH(uint256 indexed tokenId, address indexed to, uint256 amount);
    event DepositedERC20(uint256 indexed tokenId, address indexed token, address indexed from, uint256 amount);
    event WithdrawnERC20(uint256 indexed tokenId, address indexed token, address indexed to, uint256 amount);
    event GoalSet(uint256 indexed tokenId, uint256 goalWei);
    event UnlockSet(uint256 indexed tokenId, uint256 unlockTime);

    constructor(string memory baseURI_) ERC721("PiggyBank NFT", "PIGGY") {
        _baseTokenURI = baseURI_;
    }

    // --------- Mint ---------
    function mint(address to, uint256 goalWei, uint256 unlockAt) external onlyOwner returns (uint256 id) {
        id = nextId++;
        _safeMint(to, id);
        if (goalWei > 0) savingsGoal[id] = goalWei;
        if (unlockAt > 0) unlockTime[id] = unlockAt;
        emit GoalSet(id, goalWei);
        emit UnlockSet(id, unlockAt);
    }

    // --------- Deposits ---------
    function depositETH(uint256 tokenId) external payable {
        require(_exists(tokenId), "token !exist");
        require(msg.value > 0, "no value");
        ethBalance[tokenId] += msg.value;
        emit DepositedETH(tokenId, msg.sender, msg.value);
    }

    function depositERC20(uint256 tokenId, address token, uint256 amount) external {
        require(_exists(tokenId), "token !exist");
        require(amount > 0, "no amount");
        IERC20(token).safeTransferFrom(msg.sender, address(this), amount);
        erc20Balance[tokenId][token] += amount;
        emit DepositedERC20(tokenId, token, msg.sender, amount);
    }

    // --------- Withdrawals (owner only) ---------
    function withdrawETH(uint256 tokenId, uint256 amount) external nonReentrant {
        require(ownerOf(tokenId) == msg.sender, "not owner");
        require(block.timestamp >= unlockTime[tokenId], "locked");
        require(amount <= ethBalance[tokenId], "insufficient");
        ethBalance[tokenId] -= amount;
        (bool ok, ) = msg.sender.call{value: amount}("");
        require(ok, "transfer failed");
        emit WithdrawnETH(tokenId, msg.sender, amount);
    }

    function withdrawERC20(uint256 tokenId, address token, uint256 amount) external nonReentrant {
        require(ownerOf(tokenId) == msg.sender, "not owner");
        require(block.timestamp >= unlockTime[tokenId], "locked");
        require(amount <= erc20Balance[tokenId][token], "insufficient");
        erc20Balance[tokenId][token] -= amount;
        IERC20(token).safeTransfer(msg.sender, amount);
        emit WithdrawnERC20(tokenId, token, msg.sender, amount);
    }

    // --------- Settings (owner of token) ---------
    function setSavingsGoal(uint256 tokenId, uint256 goalWei) external {
        require(ownerOf(tokenId) == msg.sender, "not owner");
        savingsGoal[tokenId] = goalWei;
        emit GoalSet(tokenId, goalWei);
    }

    // Allow extending lock, never shortening (safety against tricking buyers)
    function extendUnlock(uint256 tokenId, uint256 newUnlockTime) external {
        require(ownerOf(tokenId) == msg.sender, "not owner");
        require(newUnlockTime >= unlockTime[tokenId], "cannot reduce lock");
        unlockTime[tokenId] = newUnlockTime;
        emit UnlockSet(tokenId, newUnlockTime);
    }

    // Receive plain ETH tips to a token via calldata tokenId in memo-less txs is tricky;
    // we keep deposits explicit via depositETH(tokenId).

    // --------- Metadata ---------
    function _baseURI() internal view override returns (string memory) {
        return _baseTokenURI;
    }

    function setBaseURI(string calldata newBase) external onlyOwner {
        _baseTokenURI = newBase;
    }
}
```

### Notes & safeguards

* **ReentrancyGuard** + checks-effects-interactions on withdrawals.
* **Balances are token-bound**, not address-bound → transferring the NFT transfers the vault.
* **Lock extension only** to prevent reducing a lock right before resale.
* Consider **royalty-free** for this contract (it’s a savings tool, not art), or add EIP-2981 if desired.
* Deploy on a low-fee L2 (Base / Polygon / Arbitrum).

---

## Minimal Frontend Flow (React + viem/ethers)

1. Connect wallet.
2. Select tokenId.
3. Deposit ETH to `depositETH(tokenId)` with `value`.
4. Show live balances (`ethBalance(tokenId)`).
5. Withdraw via `withdrawETH(tokenId, amount)` (only owner).

```tsx
// Pseudocode with viem
import { createPublicClient, createWalletClient, http, parseEther } from "viem";
import { base } from "viem/chains";
import PiggyBankABI from "./PiggyBankNFT.json";

const CONTRACT = "0xYourContract";

const publicClient = createPublicClient({ chain: base, transport: http() });

async function depositETH(tokenId: bigint, ethAmountStr: string, account, walletClient) {
  const value = parseEther(ethAmountStr);
  const hash = await walletClient.writeContract({
    account,
    address: CONTRACT,
    abi: PiggyBankABI,
    functionName: "depositETH",
    args: [tokenId],
    value
  });
  await publicClient.waitForTransactionReceipt({ hash });
}

async function getEthBalance(tokenId: bigint) {
  return await publicClient.readContract({
    address: CONTRACT,
    abi: PiggyBankABI,
    functionName: "ethBalance",
    args: [tokenId]
  });
}
```

---

## UX ideas to make it delightful

* **“Top-Up” button** on the NFT page that opens a modal to deposit ETH/USDC.
* **Progress ring** toward the **savings goal** (updates from `savingsGoal[tokenId]`).
* **Gift link**: prefill tokenId and amount in a deep link for friends to deposit.
* **Activity feed**: show `DepositedETH`/`WithdrawnETH` events.
* **Optional time-lock** for “do not touch until X date” savings.

---

## Deployment checklist

* Chain: **Base** testnet → mainnet.
* Verify contract on explorer.
* Set `baseURI` to a metadata service that reads on-chain balances (or renders SVG with balance).
* Add a simple **terms & risks** note (it’s a smart contract; withdrawals require gas; no interest; no custody).

---

If you want, I can also:

* add **permit** (EIP-2612) deposit support for gas-efficient ERC-20s,
* generate a **Next.js** starter with connect kit and the deposit UI,
* or extend with an **interest-bearing vault** (e.g., ERC-4626 wrapper) for yield.
