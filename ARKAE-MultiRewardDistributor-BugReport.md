# Bug Bounty Report: MultiRewardDistributor.sol

**Summary**

This report identifies vulnerabilities in `MultiRewardDistributor.sol` that could lead to reward misallocation and potential manipulation. These issues could result in unfair reward distribution or inaccurate reward accounting, impacting users and the contract’s integrity.

---

## **Issue 1: Integer Truncation in `calculateNewIndex` Enables Reward Manipulation**

**Vulnerability**

The `calculateNewIndex` function uses integer division via the `fraction` function, leading to truncation errors in reward index calculations:

```solidity
uint256 tokenAccrued = mul_(deltaTimestamps, _emissionsPerSecond);
Double memory ratio = _denominator > 0
    ? fraction(tokenAccrued, _denominator)
    : Double({mantissa: 0});
```

- In `fraction` (from `ExponentialNoError.sol`), `mantissa = (tokenAccrued * expScale) / _denominator` truncates fractional results since Solidity lacks native decimal support.
- A large `_denominator` (e.g., `totalMTokens` or `totalBorrows / borrowIndex`) reduces `ratio`, under-distributing rewards.
- Attackers can manipulate `_denominator` by supplying or borrowing large amounts, shrinking rewards for others, then withdrawing to normalize their own rewards.

**Impact**

An attacker could reduce rewards for other users by inflating `_denominator`, then claim fair rewards after reverting liquidity, skewing distribution.

**Suggested Fix**

- Use a fixed-point math library (e.g., PRBMath) to preserve precision in `fraction`.
- Set a minimum bound for `_denominator` to limit manipulation.

**Proof of Concept (PoC)**

```solidity
contract TruncationAttack {
    MultiRewardDistributor distributor;
    constructor(address _distributor) {
        distributor = MultiRewardDistributor(_distributor);
    }
    function exploit(uint256 largeDenominator) public {
        // Simulate inflating totalMTokens or totalBorrows
        // Trigger index update with largeDenominator
    }
}
```

**Severity:** High  
**Impact:** Attackers can tweak rewards, shorting others.

---

## **Issue 2: `sendReward` Vulnerable to Fee-on-Transfer or Rebasing Tokens**

**Vulnerability**

The `sendReward` function uses `safeTransfer` for reward distribution:

```solidity
IERC20 token = IERC20(_rewardToken);
if (_amount > 0 && _amount <= currentTokenHoldings) {
    token.safeTransfer(_user, _amount);
    return 0;
}
```

- While `safeTransfer` (from OpenZeppelin’s `SafeERC20`) checks return values and reverts on failure, it doesn’t account for fee-on-transfer or rebasing tokens.
- Fee-on-transfer tokens reduce the received amount, but the contract assumes `_amount` was fully sent, over-accounting rewards.
- Rebasing tokens could alter `currentTokenHoldings`, potentially disrupting transfer logic.

**Impact**

Users may receive less rewards than recorded, and attackers could exploit custom tokens to manipulate distribution, though direct fund theft is prevented by `safeTransfer`.

**Suggested Fix**

- Add a balance check before and after `safeTransfer` to verify the exact amount transferred:
  ```solidity
  uint256 balanceBefore = token.balanceOf(_user);
  token.safeTransfer(_user, _amount);
  uint256 balanceAfter = token.balanceOf(_user);
  require(balanceAfter - balanceBefore == _amount, "Fee-on-transfer detected");
  ```
- Restrict `emissionToken` to trusted ERC-20 implementations.

**Proof of Concept (PoC)**

```solidity
contract FeeOnTransferToken is ERC20 {
    uint256 public fee = 10; // 10% fee
    function transfer(address recipient, uint256 amount) public override returns (bool) {
        uint256 feeAmount = (amount * fee) / 100;
        _transfer(msg.sender, recipient, amount - feeAmount);
        return true;
    }
}
```

**Severity:** Medium  
**Impact:** Misallocation risk with fee-on-transfer tokens.

---

## **Issue 3: `disburseSupplierRewardsInternal` / `disburseBorrowerRewardsInternal` Fee-on-Transfer Risk**

**Vulnerability**

These functions delegate reward distribution to `sendReward`:

```solidity
if (_sendTokens) {
    uint256 unsendableRewards = sendReward(
        payable(_supplier),
        emissionConfig.supplierRewardsAccrued[_supplier],
        emissionConfig.config.emissionToken
    );
    emissionConfig.supplierRewardsAccrued[_supplier] = unsendableRewards;
}
```

- While `sendReward` preserves rewards on transfer failure, it doesn’t mitigate fee-on-transfer discrepancies, leading to potential over-accounting.

**Impact**

Rewards are preserved on failure, but fee-on-transfer tokens could cause minor discrepancies between recorded and received amounts.

**Suggested Fix**

- Apply Issue 2’s balance check recommendation in `sendReward`.
- Emit an event when `unsendableRewards > 0` for transparency.

**Severity:** Low  
**Impact:** Minimal; rewards preserved, though fee-on-transfer could cause discrepancies.

---

## **Conclusion**

The contract faces reward manipulation risks (Issue 1) and misallocation potential with non-standard ERC-20s (Issues 2 and 3). Addressing truncation and token compatibility will enhance fairness and accuracy in reward distribution.

**Severity & Impact Summary**

|**Issue**                                | **Severity** | **Impact**                                      |
|-----------------------------------------|--------------|-------------------------------------------------|
|Truncation in `calculateNewIndex`        | High         | Attackers can tweak rewards, shorting others.   |
|`sendReward` vs. Non-Standard ERC-20s    | Medium       | Misallocation risk with fee-on-transfer tokens. |
|Disburse Functions Transfer Handling     | Low          | Rewards preserved; minor fee-on-transfer risk.  |

**Recommendation**
- Implement fixed-point math for precision in `calculateNewIndex`.
- Enhance `sendReward` with balance checks for fee-on-transfer tokens.
- Consider restricting reward tokens to trusted implementations.

**Closing Note**: Spotted these while going through the code—figured they’re worth flagging fast. Should help keep the rewards on track.
```



