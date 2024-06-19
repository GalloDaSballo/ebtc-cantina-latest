# [Last CDP cannot be closed and user cannot recover `~2.2 stETH` from the CDP](https://github.com/cantinasec/review-badgerdao/issues/12)

> state: **open** opened by: **StErMi** on: **8/10/2023**

**Context:** [CdpManagerStorage.sol#L500-L505](https://github.com/Badger-Finance/ebtc/blob/b2f641aa20615978544547e41a4c2be642252ade/packages/contracts/contracts/CdpManagerStorage.sol#L500-L505)

**Description:** The system is designed to **not allow** the closure and removal of the last active CDP. This required and applied each time an operation (close, redeem, liquidate) tries to close a CDP.

Because of this check, that the last CDP can only be "adjusted" by the user. The user will be able to: 

- Repay at max `cdp.debt - 1 wei` of debt (repaying all the debt will revert the adjust operation).
- Withdraw at max `cdp.coll - 2 ether` of collateral (withdrawing more than that would revert because of the TCR check or because of the `MIN_NET_COLL` check).

The result is that the user will not be able to withdraw the `MIN_NET_COLL` amount of collateral and the `LIQUIDATOR_REWARD` amount of collateral provided at the creation of the CDP for a total of `~2.2 stETH` (the final amount of `stETH` depends on the conversion between `stETH shares` and `stETH` at the moment of the operation):

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.17;
import "forge-std/Test.sol";
import {eBTCBaseInvariants} from "./BaseInvariants.sol";

contract FastCloseTest is eBTCBaseInvariants {
    uint256 public constant MCR = 1100000000000000000; // 110%
    uint256 public constant DECIMAL_PRECISION = 1 ether;
    uint256 public constant MIN_COLLAERAL_SIZE = 2 ether;
    uint256 public constant GAS_STIPEND = 0.2 ether;

    address user1;
    address user2;
    bytes32 cdp1;
    bytes32 cdp2;

    function setUp() public override {
        super.setUp();

        // current value of the shares on lido (more or less)
        // collateral.setEthPerShare(1 ether);
        // priceFeedMock.setPrice((1 ether * DECIMAL_PRECISION) / 15 ether);

        connectCoreContracts();
        connectLQTYContractsToCore();
    }

    function testCloseLastCdpRevert() public {
        // Assume we are at a point where some users have opened the CDP, redempted, liquidated, closed and so on
        // We are at a point where there is only a user with an open CDP
        // The system is designed to revert always when a CDP is being closed but it's the last one active
        // This mean that the last CDP cannot be closed and the user won't be able to remove the MIN_NET_COLL
        // and LIQUIDATOR_REWARD for a total of ~2.2stETH (it depends on the share value at closing time)

        // setup the test
        address payable[] memory users;
        users = _utils.createUsers(2);
        user1 = users[0];
        vm.label(user1, "user_1");
        _dealCollateralAndPrepForUse(user1);

        // user1 creates a CDP
        uint256 borrowedAmount = _utils.calculateBorrowAmount(
            100 ether,
            priceFeedMock.fetchPrice(),
            CCR
        );
        vm.prank(user1);
        cdp1 = borrowerOperations.openCdp(borrowedAmount, "hint", "hint", 100 ether + GAS_STIPEND);

        // user1 tries to close the CDP but it reverts
        vm.prank(user1);
        vm.expectRevert("CdpManager: Only one cdp in the system");
        borrowerOperations.closeCdp(cdp1);
    }

    function testAdjustLastCdpFundsLeftInProtocol() public {
        // Assume we are at a point where some users have opened the CDP, redempted, liquidated, closed and so on
        // We are at a point where there is only a user with an open CDP
        // The system is designed to revert always when a CDP is being closed but it's the last one active
        // This mean that the last CDP cannot be closed and the user won't be able to remove the MIN_NET_COLL
        // and LIQUIDATOR_REWARD for a total of ~2.2stETH (it depends on the share value at closing time)

        // setup the test
        address payable[] memory users;
        users = _utils.createUsers(2);
        user1 = users[0];
        vm.label(user1, "user_1");
        _dealCollateralAndPrepForUse(user1);

        // user1 creates a CDP
        uint256 borrowedAmount = _utils.calculateBorrowAmount(
            100 ether,
            priceFeedMock.fetchPrice(),
            CCR
        );
        vm.prank(user1);
        cdp1 = borrowerOperations.openCdp(borrowedAmount, "hint", "hint", 100 ether + GAS_STIPEND);

        // calculate the final amount of collateral and debt of the CDP
        (uint256 debt, uint256 coll, ) = cdpManager.getEntireDebtAndColl(cdp1);

        // the user cannot repay ALL the debt because otherwise it would revert
        vm.prank(user1);
        borrowerOperations.repayEBTC(cdp1, debt - 1, "", "");

        // the user cannot withdraw all the collateral. at least MIN_NET_COLL must remain in the CDP
        vm.prank(user1);
        vm.expectRevert("BorrowerOperations: Cdp's net coll must be greater than minimum");
        borrowerOperations.withdrawColl(cdp1, coll - 2 ether + 1, "", "");

        vm.prank(user1);
        borrowerOperations.withdrawColl(cdp1, coll - 2 ether, "", "");

        // user only have 1 wei of eBTC in the balance
        assertEq(eBTCToken.balanceOf(user1), 1);

        // active pool still have the remaining stETH of the user (MIN_NET_COLL + LIQUIDATOR_REWARD)
        assertEq(collateral.balanceOf(address(activePool)), 2.2 ether);
    }

    function testDAOCDP() public {
        // one possible solution would be to have a DAO CDP that allows the last user to close the CDP
        // the problem with this DAO CDP is that it's just a "normal" CDP so at any point in time
        // it could be liquidated or redeemed (and closed) and we would be again at the start of the problem

        // setup the test
        address payable[] memory users;
        users = _utils.createUsers(2);
        user1 = users[0];
        address DAO = users[1];
        vm.label(user1, "user_1");
        vm.label(DAO, "DAO");
        _dealCollateralAndPrepForUse(user1);
        _dealCollateralAndPrepForUse(DAO);

        // DAO creates a CDP
        uint256 borrowedAmount = _utils.calculateBorrowAmount(
            100 ether,
            priceFeedMock.fetchPrice(),
            CCR
        );

        console.log("DAO open CDP");
        vm.prank(DAO);
        bytes32 cdpDAO = borrowerOperations.openCdp(
            borrowedAmount,
            "hint",
            "hint",
            100 ether + GAS_STIPEND
        );

        // user1 creates a CDP
        borrowedAmount = _utils.calculateBorrowAmount(100 ether, priceFeedMock.fetchPrice(), CCR);
        console.log("user1 open CDP");
        vm.prank(user1);
        cdp1 = borrowerOperations.openCdp(borrowedAmount, "hint", "hint", 100 ether + GAS_STIPEND);

        // user now can close it, the DAO CDP cannot be closed
        vm.prank(user1);
        borrowerOperations.closeCdp(cdp1);
    }
}
```

**Recommendation:** BadgerDAO could consider opening a healthy CDP to allow the last CDP (of the user) to be correctly closed. BadgerDAO needs to consider that this "DAO CDP" is anyway considered as a "normal CDP" and can be at any time liquidated or redeemed if the platform allows it. If this happens, we would return to the start of the scenario, where the last user won't be able to correctly close the CDP.

**BadgerDAO:** We agree with the finding and will ensure the last CDP is provided by the DAO.

**Cantina:** Just to be clear; you will "plug in" the last CDP only when it is needed, right? Because otherwise you have to maintain it during the whole life cycle of the protocol. By "maintaining it", I mean that you need to ensure that it's always healthy, and it does not get redeemed from (and closed).

**BadgerDAO:** Acknowledged.

**Cantina:** Acknowledged.

### Comments

---
> from: [**GalloDaSballo**](https://github.com/cantinasec/review-badgerdao/issues/12#issuecomment-1680081340) on: **8/16/2023**

We agree with the finding and will ensure the last CDP is provided by the DAO
---
> from: [**StErMi**](https://github.com/cantinasec/review-badgerdao/issues/12#issuecomment-1744274499) on: **10/3/2023**

@GalloDaSballo just to be clear: you will "plugin" the last CDP only when it is needed, right? Because otherwise you have to maintain it during the whole lifecycle of the protocol. For "maintaining it" I mean that you need to ensure that it's always healthy, and it does not get redeemed from (and closed)
