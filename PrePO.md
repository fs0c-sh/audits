## [M-01] The withdraw limits set in `WithdrawHook.sol` can be bypassed during the first withdrawal.

### Impact

The contract `WithdrawHook.sol` sets a withdrawal limit for its users, but this withdrawal limit can be easily bypassed by the user.

The following is the `hook` function that will be called by `Collateral.sol` contract when a withdrawal is made. 

```solidity
function hook(
    address _sender,
    uint256 _amountBeforeFee,
    uint256 _amountAfterFee
  ) external override onlyCollateral {
    require(withdrawalsAllowed, "withdrawals not allowed");
    if (lastGlobalPeriodReset + globalPeriodLength < block.timestamp) {
      lastGlobalPeriodReset = block.timestamp;
      globalAmountWithdrawnThisPeriod = _amountBeforeFee;
    } else {
      require(globalAmountWithdrawnThisPeriod + _amountBeforeFee <= globalWithdrawLimitPerPeriod, "global withdraw limit exceeded");
      globalAmountWithdrawnThisPeriod += _amountBeforeFee;
    }
    if (lastUserPeriodReset + userPeriodLength < block.timestamp) {
      lastUserPeriodReset = block.timestamp;
      userToAmountWithdrawnThisPeriod[_sender] = _amountBeforeFee;
    } else {
      require(userToAmountWithdrawnThisPeriod[_sender] + _amountBeforeFee <= userWithdrawLimitPerPeriod, "user withdraw limit exceeded");
      userToAmountWithdrawnThisPeriod[_sender] += _amountBeforeFee;
    }
    depositRecord.recordWithdrawal(_amountBeforeFee);
    uint256 _fee = _amountBeforeFee - _amountAfterFee;
    if (_fee > 0) {
      collateral.getBaseToken().transferFrom(address(collateral), _treasury, _fee);
      _tokenSender.send(_sender, _fee);
    }
  }
```

When the first withdrawal is made, it would pass the if statements and just update the value of `lastGlobalPeriodReset` and `globalAmountWithdrawnThisPeriod` for global checks and `lastUserPeriodReset` and `userToAmountWithdrawnThisPeriod` for the user checks as the value of `lastGlobalPeriodReset` and `lastUserPeriodReset` would be initially zero. 

So directly passing the value greater than both `globalWithdrawLimitPerPeriod` and `userWithdrawLimitPerPeriod` would bypass the limits as the function won’t get into the else statements for both the checks, hence both the require statements in else condition won’t be run.

### POC

I’ve written the POC in foundry. The setup would be needed first to run the POC. Here is a high level overview of how to set it up:

1. `forge init`
2. `forge install OpenZeppelin/openzeppelin-contracts`
3. `forge install openzeppelin/contracts-upgradeable`
4. Copy all the files from here ([https://github.com/prepo-io/prepo-monorepo/tree/feat/2022-12-prepo/apps/smart-contracts/core/contracts](https://github.com/prepo-io/prepo-monorepo/tree/feat/2022-12-prepo/apps/smart-contracts/core/contracts)) to `src/` (which was created during forge init)
5. Copy all files from `prepo-shared-contracts` to `/lib/prepo-shared-contracts` 
6. Create a remappings.txt file will the correct remappings.

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.6;

import "forge-std/Test.sol";
import "../src/test/TestERC20.sol";
import "../src/Collateral.sol";
import "../src/DepositHook.sol";
import "../src/WithdrawHook.sol";
import "../src/DepositRecord.sol";
import "../src/TokenSender.sol";

contract CounterTest is Test {
    TestERC20 public mockerc20;
    TestERC20 public outputtoken;
    Collateral public collateral;
    DepositHook public deposithook;
    WithdrawHook public withdrawhook;
    DepositRecord public depositrecord;
    TokenSender public tokensender;

    // collateral roles
    bytes32 public constant MANAGER_WITHDRAW_ROLE = keccak256("Collateral_managerWithdraw(uint256)");
    bytes32 public constant SET_MANAGER_ROLE = keccak256("Collateral_setManager(address)");
    bytes32 public constant SET_DEPOSIT_FEE_ROLE = keccak256("Collateral_setDepositFee(uint256)");
    bytes32 public constant SET_WITHDRAW_FEE_ROLE = keccak256("Collateral_setWithdrawFee(uint256)");
    bytes32 public constant SET_DEPOSIT_HOOK_ROLE = keccak256("Collateral_setDepositHook(ICollateralHook)");
    bytes32 public constant SET_WITHDRAW_HOOK_ROLE = keccak256("Collateral_setWithdrawHook(ICollateralHook)");
    bytes32 public constant SET_MANAGER_WITHDRAW_HOOK_ROLE = keccak256("Collateral_setManagerWithdrawHook(ICollateralHook)");

    // withdraw roles
    bytes32 public constant SET_COLLATERAL_ROLE = keccak256("WithdrawHook_setCollateral(address)");
    bytes32 public constant SET_DEPOSIT_RECORD_ROLE = keccak256("WithdrawHook_setDepositRecord(address)");
    bytes32 public constant SET_WITHDRAWALS_ALLOWED_ROLE = keccak256("WithdrawHook_setWithdrawalsAllowed(bool)");
    bytes32 public constant SET_GLOBAL_PERIOD_LENGTH_ROLE = keccak256("WithdrawHook_setGlobalPeriodLength(uint256)");
    bytes32 public constant SET_USER_PERIOD_LENGTH_ROLE = keccak256("WithdrawHook_setUserPeriodLength(uint256)");
    bytes32 public constant SET_GLOBAL_WITHDRAW_LIMIT_PER_PERIOD_ROLE = keccak256("WithdrawHook_setGlobalWithdrawLimitPerPeriod(uint256)");
    bytes32 public constant SET_USER_WITHDRAW_LIMIT_PER_PERIOD_ROLE = keccak256("WithdrawHook_setUserWithdrawLimitPerPeriod(uint256)");
    bytes32 public constant SET_TREASURY_ROLE = keccak256("WithdrawHook_setTreasury(address)");
    bytes32 public constant SET_TOKEN_SENDER_ROLE = keccak256("WithdrawHook_setTokenSender(ITokenSender)");

    //deposit roles
    bytes32 public constant SET_COLLATERAL_ROLE1 = keccak256("DepositHook_setCollateral(address)");
    bytes32 public constant SET_DEPOSIT_RECORD_ROLE1 = keccak256("DepositHook_setDepositRecord(address)");
    bytes32 public constant SET_DEPOSITS_ALLOWED_ROLE1 = keccak256("DepositHook_setDepositsAllowed(bool)");
    bytes32 public constant SET_ACCOUNT_LIST_ROLE1 = keccak256("DepositHook_setAccountList(IAccountList)");
    bytes32 public constant SET_REQUIRED_SCORE_ROLE1 = keccak256("DepositHook_setRequiredScore(uint256)");
    bytes32 public constant SET_COLLECTION_SCORES_ROLE1 = keccak256("DepositHook_setCollectionScores(IERC721[],uint256[])");
    bytes32 public constant REMOVE_COLLECTIONS_ROLE1 = keccak256("DepositHook_removeCollections(IERC721[])");
    bytes32 public constant SET_TREASURY_ROLE1 = keccak256("DepositHook_setTreasury(address)");
    bytes32 public constant SET_TOKEN_SENDER_ROLE1 = keccak256("DepositHook_setTokenSender(ITokenSender)");

    bytes32 public constant SET_GLOBAL_NET_DEPOSIT_CAP_ROLE = keccak256("DepositRecord_setGlobalNetDepositCap(uint256)");
    bytes32 public constant SET_USER_DEPOSIT_CAP_ROLE = keccak256("DepositRecord_setUserDepositCap(uint256)");
    bytes32 public constant SET_ALLOWED_HOOK_ROLE = keccak256("DepositRecord_setAllowedHook(address)");

    function setUp() public {
        mockerc20 = new TestERC20("TEST-Token","TEST",18);
        outputtoken = new TestERC20("Output Token","OTOKEN",18);
        collateral = new Collateral(mockerc20,18);
        deposithook = new DepositHook();
        withdrawhook = new WithdrawHook();
        depositrecord = new DepositRecord(50000 ether,10000 ether);
        tokensender = new TokenSender(outputtoken);
        collateral.initialize("TEST-Token", "TEST");
        collateral.grantRole(SET_MANAGER_ROLE, address(this));
        collateral.acceptRole(SET_MANAGER_ROLE);
        collateral.grantRole(SET_DEPOSIT_FEE_ROLE, address(this));
        collateral.acceptRole(SET_DEPOSIT_FEE_ROLE);
        collateral.grantRole(SET_WITHDRAW_FEE_ROLE, address(this));
        collateral.acceptRole(SET_WITHDRAW_FEE_ROLE);
        collateral.grantRole(SET_DEPOSIT_HOOK_ROLE, address(this));
        collateral.acceptRole(SET_DEPOSIT_HOOK_ROLE);
        collateral.grantRole(SET_WITHDRAW_HOOK_ROLE, address(this));
        collateral.acceptRole(SET_WITHDRAW_HOOK_ROLE);
        collateral.grantRole(SET_MANAGER_WITHDRAW_HOOK_ROLE, address(this));
        collateral.acceptRole(SET_MANAGER_WITHDRAW_HOOK_ROLE);
        

        // collateral.setDepositHook(deposithook);
        deposithook.grantRole(SET_COLLATERAL_ROLE1, address(this));
        deposithook.acceptRole(SET_COLLATERAL_ROLE1);
        deposithook.grantRole(SET_DEPOSIT_RECORD_ROLE1, address(this));
        deposithook.acceptRole(SET_DEPOSIT_RECORD_ROLE1);
        deposithook.grantRole(SET_DEPOSITS_ALLOWED_ROLE1, address(this));
        deposithook.acceptRole(SET_DEPOSITS_ALLOWED_ROLE1);
        deposithook.grantRole(SET_ACCOUNT_LIST_ROLE1, address(this));
        deposithook.acceptRole(SET_ACCOUNT_LIST_ROLE1);
        deposithook.grantRole(SET_REQUIRED_SCORE_ROLE1, address(this));
        deposithook.acceptRole(SET_REQUIRED_SCORE_ROLE1);
        deposithook.grantRole(SET_COLLECTION_SCORES_ROLE1, address(this));
        deposithook.acceptRole(SET_COLLECTION_SCORES_ROLE1);
        deposithook.grantRole(REMOVE_COLLECTIONS_ROLE1, address(this));
        deposithook.acceptRole(REMOVE_COLLECTIONS_ROLE1);
        deposithook.grantRole(SET_TREASURY_ROLE1, address(this));
        deposithook.acceptRole(SET_TREASURY_ROLE1);
        deposithook.grantRole(SET_TOKEN_SENDER_ROLE1, address(this));
        deposithook.acceptRole(SET_TOKEN_SENDER_ROLE1);
        deposithook.setCollateral(collateral);
        deposithook.setDepositsAllowed(true);
        deposithook.setDepositRecord(depositrecord);

        collateral.setWithdrawHook(withdrawhook);
        withdrawhook.grantRole(SET_COLLATERAL_ROLE, address(this));
        withdrawhook.acceptRole(SET_COLLATERAL_ROLE);
        withdrawhook.grantRole(SET_DEPOSIT_RECORD_ROLE, address(this));
        withdrawhook.acceptRole(SET_DEPOSIT_RECORD_ROLE);
        withdrawhook.grantRole(SET_WITHDRAWALS_ALLOWED_ROLE, address(this));
        withdrawhook.acceptRole(SET_WITHDRAWALS_ALLOWED_ROLE);
        withdrawhook.grantRole(SET_GLOBAL_PERIOD_LENGTH_ROLE, address(this));
        withdrawhook.acceptRole(SET_GLOBAL_PERIOD_LENGTH_ROLE);
        withdrawhook.grantRole(SET_USER_PERIOD_LENGTH_ROLE, address(this));
        withdrawhook.acceptRole(SET_USER_PERIOD_LENGTH_ROLE);
        withdrawhook.grantRole(SET_GLOBAL_WITHDRAW_LIMIT_PER_PERIOD_ROLE, address(this));
        withdrawhook.acceptRole(SET_GLOBAL_WITHDRAW_LIMIT_PER_PERIOD_ROLE);
        withdrawhook.grantRole(SET_USER_WITHDRAW_LIMIT_PER_PERIOD_ROLE, address(this));
        withdrawhook.acceptRole(SET_USER_WITHDRAW_LIMIT_PER_PERIOD_ROLE);
        withdrawhook.grantRole(SET_TREASURY_ROLE, address(this));
        withdrawhook.acceptRole(SET_TREASURY_ROLE);
        withdrawhook.grantRole(SET_TOKEN_SENDER_ROLE, address(this));
        withdrawhook.acceptRole(SET_TOKEN_SENDER_ROLE);
        withdrawhook.setDepositRecord(depositrecord);
        withdrawhook.setCollateral(collateral);
        withdrawhook.setWithdrawalsAllowed(true);
        withdrawhook.setTokenSender(tokensender);
        withdrawhook.setGlobalPeriodLength(20);
        withdrawhook.setUserPeriodLength(10);
        withdrawhook.setGlobalWithdrawLimitPerPeriod(300 ether); //[1] global limit set to 300
        withdrawhook.setUserWithdrawLimitPerPeriod(250 ether); //[2] user limit set to 250

        depositrecord.grantRole(SET_GLOBAL_NET_DEPOSIT_CAP_ROLE, address(this));
        depositrecord.acceptRole(SET_GLOBAL_NET_DEPOSIT_CAP_ROLE);
        depositrecord.grantRole(SET_USER_DEPOSIT_CAP_ROLE, address(this));
        depositrecord.acceptRole(SET_USER_DEPOSIT_CAP_ROLE);
        depositrecord.grantRole(SET_ALLOWED_HOOK_ROLE, address(this));
        depositrecord.acceptRole(SET_ALLOWED_HOOK_ROLE);
        depositrecord.setAllowedHook(address(withdrawhook), true);

    }

    function testdeposit() public{
        address alice;
        alice = vm.addr(1);
        vm.warp(1670777395);

        mockerc20.mint(address(alice), 1000 ether);
        vm.prank(alice);
        mockerc20.approve(address(collateral),  1000 ether);

        vm.prank(alice);
        collateral.deposit(address(alice), 500 ether);

        emit log_named_decimal_uint( "Collateral Tokens: ", collateral.balanceOf(alice), 18);
        
        vm.prank(alice);
        collateral.withdraw(350 ether); // withdrawal of 350 works
    }
}
```

As shown in the code above at comment [1] the global withrawal limit is set to `300` , at [2] the user withdrawal limit is set to `250` , but at [3] the withdraw of 350 works. 

The withdrawal won’t work if the user withdraws the sum in more than 2 different transactions (eg: first withdrawal of 200 and second of 150) as the bug is triggered when the user withdraws just the first time.

### Recommended Mitigations Steps

One way would be to add another check and see if the `_amountBeforeFee` is greater than user/global withdrawal limit and revert if it is greater.

## [M-02] Possible DOS when calling `deposit` in `Collateral.sol`

### Impact

The `deposit` function calls the `hook` function which checks if the `_sender` passes score requirement by calling the `_satisfiesScoreRequirement` if `_sender` is not included in `_accountList` . 

The check  `_satisfiesScoreRequirement` runs in an unbounded loop :

```solidity
function _satisfiesScoreRequirement(address account) internal view virtual returns (bool) {
    return _requiredScore == 0 || getAccountScore(account) >= _requiredScore;
  }

function getAccountScore(address account) public view virtual override returns (uint256) {
    uint256 score;
    uint256 numCollections = _collectionToScore.length();
    for (uint256 i = 0; i < numCollections; ) {
      (address collection, uint256 collectionScore) = _collectionToScore.at(i);
      score += IERC721(collection).balanceOf(account) > 0 ? collectionScore : 0;
      unchecked {
        ++i;
      }
    }
    return score;
  }
```

If a lot of collections are added, the loop would eventually run out of gas and the call would revert, thus users won’t be able to deposit in the collateral contract.

### Recommendation

A new mapping can be made which updates the score of an account, when calling `setCollectionScores` . 

## [M-03] Redeem should revert if the `TokenSender.sol` doesn’t have enough outputtoken to reimburse to users.

According to the comments in the code of `RedeemHook.hook` function : 

```markdown
Once a market has ended, users can directly settle their positions with the market contract. Because users might call `redeem()`, a fee might be taken and will be reimbursed using `_tokenSender`.
```

The code flow goes like this :

```markdown
redeem() [PrePOMarket.sol] -> hook [RedeemHook.sol] -> send [TokenSender.sol]
```

Here is send function from tokensender is supposted to send the outputtoken to the caller:

```solidity
function send(address recipient, uint256 unconvertedAmount) external override onlyAllowedMsgSenders {
    uint256 scaledPrice = (_price.get() * _priceMultiplier) / MULTIPLIER_DENOMINATOR;
    if (scaledPrice <= _scaledPriceLowerBound) return;
    uint256 outputAmount = (unconvertedAmount * _outputTokenDecimalsFactor) / scaledPrice;
    if (outputAmount == 0) return;
    if (outputAmount > _outputToken.balanceOf(address(this))) return;
    _outputToken.transfer(recipient, outputAmount);
  }
```

But in case where `outputAmount > _outputToken.balanceOf(address(this))` the code simply returns, where it should have reverted. 

Impact: User would not be re-imbursed the amount they were supposed to get. 

POC: [https://github.com/prepo-io/prepo-monorepo/blob/49a7ed94272db013245d9364e69be713a8aef0a2/apps/smart-contracts/core/contracts/TokenSender.sol#L41](https://github.com/prepo-io/prepo-monorepo/blob/49a7ed94272db013245d9364e69be713a8aef0a2/apps/smart-contracts/core/contracts/TokenSender.sol#L41)

Recommendation:

Add a revert statement here.