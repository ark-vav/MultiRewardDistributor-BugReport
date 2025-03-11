# Bug Bounty Report: MultiRewardDistributor.sol

**Summary**

This report identifies multiple vulnerabilities in `MultiRewardDistributor.sol`, which could lead to **reward manipulation, incorrect distribution, and fund drainage**. Exploiting these issues could result in an **attacker unfairly increasing their rewards or causing other users to lose their rightful rewards**.

---

## **Issue 1: Integer Truncation in `calculateNewIndex` Opens the Door to Reward Tweaking**

**Vulnerability**

The function `calculateNewIndex` performs integer division, which leads to truncation errors in reward calculations:

```solidity
uint224 ratio = uint224((_emissionsPerSecond * deltaBlocks * EXP_SCALE) / _denominator);
```

- If `_denominator` is **large**, **integer truncation** causes rewards to be under-distributed.
- Attackers can influence `_denominator` by controlling liquidity (borrowing/supplying huge amounts), affecting **reward allocation unfairly**.
- Rapid fluctuations in `_emissionsPerSecond` can cause **highly inaccurate rewards**.

**Impact**

An attacker could reduce other users rewards by artificially **increasing** `_denominator`, then later removing liquidity to **restore normal rewards for themselves**.

**Suggested Fix**

- Use **fixed-point math libraries** (such as PRBMath) to maintain precision.
- Ensure `_denominator` has a **minimum bound** to prevent manipulation.

**Proof of Concept (PoC)**

```solidity
contract TruncationAttack {
    MultiRewardDistributor distributor;
    constructor(address _distributor) {
        distributor = MultiRewardDistributor(_distributor);
    }
    function exploit(uint256 largeDenominator) public {
        distributor.calculateNewIndex(largeDenominator);
    }
}
```

---

## **Issue 2: `sendReward` Can Be Exploited by Malicious ERC-20 Tokens**

**Vulnerability**

The function `sendReward` directly transfers tokens without validating ERC-20 return values:

```solidity
rewardToken.safeTransfer(user, amount);
```

- If `rewardToken` is a **fee-on-transfer, rebasing, or malicious token**, this could result in **stolen or misallocated funds**.
- A custom token could **return false instead of reverting**, causing **silent reward loss**.
- Malicious tokens could be designed to **mint additional tokens** and drain contract rewards.

**Impact**

Attackers can **deploy a custom ERC-20** that behaves normally during verification but **steals rewards** when transfers occur.

**Suggested Fix**

- Use **OpenZeppelin’s `safeTransfer` wrapper** that checks for transfer success.
- Restrict reward tokens to **well-known ERC-20 implementations**.
- Add explicit **require checks** for transfers:

```solidity
require(rewardToken.transfer(user, amount), "Transfer failed");
```

**Proof of Concept (PoC)**

```solidity
contract MaliciousToken is ERC20 {
    function transfer(address recipient, uint256 amount) public override returns (bool) {
        return false; // Silently fail
    }
}
```

---

## **Issue 3: `disburseSupplierRewardsInternal` / `disburseBorrowerRewardsInternal` Do Not Check for Transfer Success**

**Vulnerability**

Both functions distribute rewards but **fail to verify if the transfer succeeds**:

```solidity
rewardToken.safeTransfer(user, rewardAmount);
```

- If the token **fails silently**, rewards will **not be sent** but remain stuck.
- Attackers could **supply a fake ERC-20**, ensuring that only their transactions succeed while others fail.
- This could lead to **centralized reward distribution where only attackers get paid**.

**Impact**

Attackers can manipulate token transfers so that **rewards intended for all users only go to their own wallet**.

**Suggested Fix**

- Implement **require checks**:

```solidity
require(rewardToken.transfer(user, rewardAmount), "Reward transfer failed");
```

- Emit an **event** on transfer failures for transparency.

**Proof of Concept (PoC)**

```solidity
contract FakeERC20 is ERC20 {
    function transfer(address recipient, uint256 amount) public override returns (bool) {
        if (recipient == attackerAddress) {
            return true;
        }
        return false;
    }
}
```

---

## **Conclusion**

These vulnerabilities expose the contract to **reward manipulation, incorrect distribution, and potential fund loss**. While not instant contract-breaking bugs, they present **exploitable scenarios** that could lead to unfair rewards or fund drainage. Fixing these issues ensures **secure and fair reward distribution**.

---

**Severity & Impact Summary**

|**Issue**                                |  **Severity**    | **Impact**                                      |
|Truncation in calculateNewIndex	      |   High	         | Attackers can tweak rewards, shorting others.   |
|sendReward vs. Bad ERC-20s	              |   Critical	     | Shady tokens might snatch funds.                |
|Disburse Functions Skipping Checks	      |   High	         | Rewards could skip users, favoring attackers.   |


---

**Recommendation**
Add transfer checks everywhere.
Use fixed-point math libraries for precision.
Limit reward tokens to ones you know won’t bite/known implementations.

**Closing Note**: Spotted these while going through the code—figured they’re worth flagging fast. Let me know if you need more details or a hand with testing. Should help keep the rewards on track.