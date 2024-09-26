# H-01 Malicious users can exploit raffle cancellation to disrupt protocol functionality

## Summary
Malicious users can exploit the `cancelRaffle` function to repeatedly cancel raffles, preventing new raffles from being created and disrupting ticket sales. This makes the protocol unusable by forcing admins to constantly lock new prizes and create new raffles.

## Vulnerability Detail
The `WinnablesTicketManager::cancelRaffle` function is designed to allow the cancellation of raffles under specific pre-conditions:
1. The raffle is not yet created, and its status is `PRIZE_LOCKED`, enabling admins to cancel raffles and unlock the prize in `WinnablesPrizeManager` in cases of misconfigured prizes.
2. The raffle period has ended, and the minimum ticket threshold has not been reached, allowing for refunds to users who purchased tickets.
```solidity
function _checkShouldCancel(uint256 raffleId) internal view {
    Raffle storage raffle = _raffles[raffleId];
    if (raffle.status == RaffleStatus.PRIZE_LOCKED) return;
    if (raffle.status != RaffleStatus.IDLE) revert InvalidRaffle();

    if (raffle.endsAt > block.timestamp) revert RaffleIsStillOpen(); 
    uint256 supply = IWinnablesTicket(TICKETS_CONTRACT).supplyOf(raffleId);
        
    if (supply > raffle.minTicketsThreshold) revert TargetTicketsReached();
}
```
However, malicious users can repeatedly cancel raffles through `WinnablesTicketManager::cancelRaffle` by exploiting the first pre-condition. Specifically, after the `ccipReceive` function triggers and the prize is locked, the cancellation check passes, allowing the raffle to be canceled:
```solidity
function cancelRaffle(address prizeManager, uint64 chainSelector, uint256 raffleId) external {
>>>    _checkShouldCancel(raffleId);
    _raffles[raffleId].status = RaffleStatus.CANCELED;
    // ...
}

function _checkShouldCancel(uint256 raffleId) internal view {
    Raffle storage raffle = _raffles[raffleId];
>>>    if (raffle.status == RaffleStatus.PRIZE_LOCKED) return;
    // ...
}
```
When a raffle is canceled, it cannot be created:
```solidity
function createRaffle(
    uint256 raffleId,
    uint64 startsAt,
    uint64 endsAt,
    uint32 minTickets,
    uint32 maxTickets,
    uint32 maxHoldings
) external onlyRole(0) {
    _checkRaffleTimings(startsAt, endsAt);
    if (maxTickets == 0) revert RaffleRequiresTicketSupplyCap();
    if (maxHoldings == 0) revert RaffleRequiresMaxHoldings();
    Raffle storage raffle = _raffles[raffleId];
>>>    if (raffle.status != RaffleStatus.PRIZE_LOCKED) revert PrizeNotLocked();

    // ...
}
```
As a result, admins are forced to lock new prizes in `WinnablePrizeManager` to create new raffles, but malicious users can continue to cancel these raffles immediately in `WinnablesTicketManagers` after the prize is locked, disrupting the protocol's functionality.

## Impact
Malicious users can repeatedly cancel raffles, preventing the creation of new raffles, stopping users from purchasing tickets, and forcing administrators to constantly create new raffles. This behavior renders the protocol unusable.

## Code Snippet
https://github.com/sherlock-audit/2024-08-winnables-raffles/blob/e8b0603f6a155c7505dacc77194ae6789d0dbe7a/public-contracts/contracts/WinnablesTicketManager.sol#L434-L436

## Tool used

Manual Review

## Recommendation
Consider restricting the ability to cancel raffles when the status is `PRIZE_LOCKED` to admins only. Below is a suggested update to the `_checkShouldCancel` function:
```diff
function _checkShouldCancel(uint256 raffleId) internal view {
    Raffle storage raffle = _raffles[raffleId];
    if (raffle.status == RaffleStatus.PRIZE_LOCKED) {
+       _checkRole(msg.sender, 0);
    };
    if (raffle.status != RaffleStatus.IDLE) revert InvalidRaffle();

    if (raffle.endsAt > block.timestamp) revert RaffleIsStillOpen(); 
    uint256 supply = IWinnablesTicket(TICKETS_CONTRACT).supplyOf(raffleId);
        
    if (supply > raffle.minTicketsThreshold) revert TargetTicketsReached();
}
```

# H-02 Failure to update `_lockedETH` during refunds causes inaccurate revenue withdrawals

## Summary
The `_lockedETH` variable is not updated during refunds, leading to an inflated value that prevents admins from correctly withdrawing future sales revenue.

## Vulnerability Detail
The `_lockedETH` variable represents the amount of ETH locked in the contract [due to ticket purchases](https://github.com/sherlock-audit/2024-08-winnables-raffles/blob/e8b0603f6a155c7505dacc77194ae6789d0dbe7a/public-contracts/contracts/WinnablesTicketManager.sol#L206). It is used to ensure that the ETH associated with current unfinished raffles is not included in the sales revenue of past raffles during withdrawals:
```solidity
/// @notice (Admin) Withdraw ETH from a canceled raffle or ticket sales
function withdrawETH() external onlyRole(0) {
    uint256 balance;
    unchecked {
        balance = address(this).balance - _lockedETH; 
    }
    _sendETH(balance, msg.sender);
}
```
However, the `_lockedETH` value is not updated when a refund occurs due to raffle cancellation:
```solidity
function refundPlayers(uint256 raffleId, address[] calldata players) external {
    Raffle storage raffle = _raffles[raffleId];
    if (raffle.status != RaffleStatus.CANCELED) revert InvalidRaffle();
    for (uint256 i = 0; i < players.length; ) {
        address player = players[i];
        uint256 participation = uint256(raffle.participations[player]);
        if (((participation >> 160) & 1) == 1) revert PlayerAlreadyRefunded(player);
        raffle.participations[player] = bytes32(participation | (1 << 160));
        uint256 amountToSend = (participation & type(uint128).max);
        _sendETH(amountToSend, player);
        emit PlayerRefund(raffleId, player, bytes32(participation));
        unchecked { ++i; }
    }
}
```
As seen above, when a refund occurs, the `_lockedETH` amount is not decremented. This leads to an inflated `_lockedETH` value, preventing the admin from withdrawing future sales revenue equal to the excess amount in `_lockedETH`. Consider the following example:
1. Raffle 1 is created with a minimum tickets threshold of `400`.
2. Buyer 1 purchases `100` tickets for `0.5 ETH`, and Buyer 2 purchases `150` tickets for `0.75 ETH`.
3. The raffle period ends without reaching the minimum tickets threshold (only 250 tickets are sold), leading to the raffle's cancellation.
4. Both Buyer 1 and Buyer 2 are refunded via `refundPlayers(1, [buyer1.address, buyer2.address])`. Note that during this transaction, `_lockedETH` is NOT updated.
5. Depending on subsequent scenarios:
    5.1 If a new raffle is created and successfully ends with sales revenue greater than `_lockedETH`, say for example `1 ETH`, the admin attempts to withdraw the `1 ETH`. However, only `0.25 ETH` will be withdrawn:
    ```solidity
    function withdrawETH() external onlyRole(0) {
        uint256 balance;
        unchecked {
            balance = address(this).balance - _lockedETH; // @audit balance = 1 ETH - 0.75 ETH = 0.25 ETH
        }
        _sendETH(balance, msg.sender);
    }
    ```
    5.2 If a new raffle is created and successfully ends with sales revenue less than `_lockedETH`say for example `0.5 ETH`, the admin's withdrawal attempt will result in an underflow, causing the withdrawal to revert because the contract does not hold sufficient balance:
    ```solidity
    function withdrawETH() external onlyRole(0) {
        uint256 balance;
        unchecked {
    >>>       balance = address(this).balance - _lockedETH; // @audit Underflow: 0.5 ETH - 0.75 ETH
        }
        _sendETH(balance, msg.sender);
    }
    ```
## Impact
Admins will be unable to withdraw the correct sales revenue, leading to financial loss.
## Code Snippet
https://github.com/sherlock-audit/2024-08-winnables-raffles/blob/e8b0603f6a155c7505dacc77194ae6789d0dbe7a/public-contracts/contracts/WinnablesTicketManager.sol#L215-L228
## Tool used

Manual Review

## Recommendation
Update `_lockedETH` during the refund process. Below is a suggested modification to the `refundPlayers` function:
```diff
function refundPlayers(uint256 raffleId, address[] calldata players) external {
    Raffle storage raffle = _raffles[raffleId];
    if (raffle.status != RaffleStatus.CANCELED) revert InvalidRaffle();
     for (uint256 i = 0; i < players.length; ) {
        address player = players[i];
        uint256 participation = uint256(raffle.participations[player]);
        if (((participation >> 160) & 1) == 1) revert PlayerAlreadyRefunded(player);
        raffle.participations[player] = bytes32(participation | (1 << 160));
        uint256 amountToSend = (participation & type(uint128).max);
        _sendETH(amountToSend, player);
+       _lockedETH -= amountToSend;
        emit PlayerRefund(raffleId, player, bytes32(participation));
        unchecked { ++i; }
    }
}
```

# H-03 Lack of parameter validation in `propagateRaffleWinner` Leads to failed prize claims on mainnet

## Summary
The `propagateRaffleWinner` function can be exploited by providing incorrect `prizeManager` and `chainSelector` parameters, leading to a situation where the raffle is marked as propagated, but the winner is unable to claim their prize on the mainnet.

## Vulnerability Detail
After a raffle is drawn and a winner is determined via Chainlink VRF, anyone can call the `propagateRaffleWinner` function to propagate the winner's information to the mainnet, allowing the winner to claim their prize:
```solidity
function propagateRaffleWinner(address prizeManager, uint64 chainSelector, uint256 raffleId) external {
    Raffle storage raffle = _raffles[raffleId];
    if (raffle.status != RaffleStatus.FULFILLED) revert InvalidRaffleStatus();
    raffle.status = RaffleStatus.PROPAGATED;
    address winner = _getWinnerByRequestId(raffle.chainlinkRequestId);

>>>    _sendCCIPMessage(prizeManager, chainSelector, abi.encodePacked(uint8(CCIPMessageType.WINNER_DRAWN), raffleId, winner));
    IWinnablesTicket(TICKETS_CONTRACT).refreshMetadata(raffleId);
    unchecked {
        _lockedETH -= raffle.totalRaised;
    }
}

function _sendCCIPMessage(
        address ccipDestAddress,
        uint64 ccipDestChainSelector,
        bytes memory data
    ) internal returns(bytes32 messageId) {
        if (ccipDestAddress == address(0) || ccipDestChainSelector == uint64(0)) { // @audit Only basic checks
            revert MissingCCIPParams();
        }

        // Send CCIP message to the desitnation contract
        IRouterClient router = IRouterClient(CCIP_ROUTER);
        LinkTokenInterface linkToken = LinkTokenInterface(LINK_TOKEN);

        Client.EVM2AnyMessage memory message = Client.EVM2AnyMessage({
            receiver: abi.encode(ccipDestAddress),
            data: data,
            tokenAmounts: new Client.EVMTokenAmount[](0),
            extraArgs: "",
            feeToken: LINK_TOKEN
        });

    uint256 fee = router.getFee(
        ccipDestChainSelector,
        message
    );
    uint256 currentLinkBalance = linkToken.balanceOf(address(this));

    if (fee > currentLinkBalance) {
        revert InsufficientLinkBalance(currentLinkBalance, fee);
    }

    messageId = router.ccipSend(
        ccipDestChainSelector,
        message
    );
}
```
As seen above, the raffle status is set to `PROPAGATED` and `_lockedETH` is decremented immediately after the function is called. However, the function does not perform adequate checks on the `prizeManager` address or the `chainSelector`. As a result, a malicious user could provide an unsupported chain or an incorrect `prizeManager` address. This would result in the transaction being marked as successful on the source chain (with the raffle status set to `PROPAGATED` and `_lockedETH` decremented), but the Prize Manager contract on the mainnet would not be notified about the raffle's winner. Consequently, the winner would be unable to claim their prize on the mainnet.
## Impact
A malicious user can exploit this vulnerability by calling `propagateRaffleWinner` with incorrect `prizeManager` or `chainSelector` values, causing the raffle to be incorrectly marked as propagated on Avalanche, while the winner would be unable to claim their prize on the mainnet.
## Code Snippet
https://github.com/sherlock-audit/2024-08-winnables-raffles/blob/e8b0603f6a155c7505dacc77194ae6789d0dbe7a/public-contracts/contracts/WinnablesTicketManager.sol#L340
## Tool used

Manual Review

## Recommendation
Consider restricting the `propagateRaffleWinner` function to be callable only by an admin, or implement additional checks to validate the `prizeManager` and `chainSelector` parameters.

# M-01 Roles can not be revoked

## Vulnerability Detail
The `setRole` function accepts a `status` parameter intended to toggle the role for the specified `user`:
```solidity
function setRole(address user, uint8 role, bool status) external virtual onlyRole(0) {
    _setRole(user, role, status);
}

function _setRole(address user, uint8 role, bool status) internal virtual {
    uint256 roles = uint256(_addressRoles[user]);
>>>    _addressRoles[user] = bytes32(roles | (1 << role));
    emit RoleUpdated(user, role, status);
}   
```
However, the current implementation does not allow for revoking a role. When the `status` parameter is set to `false`, the role is not removed; instead, it remains active for the user:
```solidity
_addressRoles[user] = bytes32(roles | (1 << role));
```

## Proof Of Concept
Copy and paste the following test case into `test/Ticket.js`:
```js
describe('Ticket behaviour', () => {
    // ...
    it('can not revoke role', async () => {
      await ticket.setRole(signers[1].address, 1, true);
      let roleBeforeRevoking = await ticket.getRoles(signers[1].address);
      await ticket.setRole(signers[1].address, 1, false);
      let roleAfterRevoking = await ticket.getRoles(signers[1].address);
      assert(roleBeforeRevoking == roleAfterRevoking)
    });
})
```
## Impact
Roles can not be revoked
## Code Snippet
https://github.com/sherlock-audit/2024-08-winnables-raffles/blob/e8b0603f6a155c7505dacc77194ae6789d0dbe7a/public-contracts/contracts/Roles.sol#L31

## Tool used

Manual Review

## Recommendation
Update `_setRole` function to handle the revocation of roles when `status` is set to false.


# M-02 Admins can influence the odds of raffles

## Summary
Admins can influence the odds of a raffle by minting additional tickets after the random word is fulfilled by Chainlink VRF. This breaks the core invariant that admins should not influence raffle odds.

## Vulnerability Detail
The contest readme explicitly states that administrators should not have any means to influence the odds of a raffle:
> The principles that must always remain true are: 
> - **Admins cannot affect the odds of a raffle**

The raffle winner is determined after the Chainlink VRF fulfills the random word, with the winner calculated based on the random word and the raffle's ticket supply using the formula `randomWord % supply`:
```solidity
function _getWinnerByRequestId(uint256 requestId) internal view returns(address) {
    RequestStatus storage request = _chainlinkRequests[requestId];
    uint256 supply = IWinnablesTicket(TICKETS_CONTRACT).supplyOf(request.raffleId);
>>>    uint256 winningTicketNumber = request.randomWord % supply;
    return IWinnablesTicket(TICKETS_CONTRACT).ownerOf(request.raffleId, winningTicketNumber);
}
```
The issue is that after the random word is fulfilled—meaning after the [fulfillRandomWords](https://github.com/sherlock-audit/2024-08-winnables-raffles/blob/e8b0603f6a155c7505dacc77194ae6789d0dbe7a/public-contracts/contracts/WinnablesTicketManager.sol#L350-L361) function is called and the random word is known. At this point, the admin can determine the random word and exploit it by calling the [WinnablesTicket::mint](https://github.com/sherlock-audit/2024-08-winnables-raffles/blob/e8b0603f6a155c7505dacc77194ae6789d0dbe7a/public-contracts/contracts/WinnablesTicket.sol#L182-L199) function to mint additional tickets. By increasing the ticket supply, the admin can manipulate the outcome of the raffle, thereby controlling who wins:
```solidity
// WinnablesTicket::mint
function mint(address to, uint256 id, uint256 amount) external onlyRole(1) {
    // ...

    unchecked {
      _balances[id][to] += amount;
>>>      _supplies[id] = startId + amount;
    }

    _ticketOwnership[id][startId] = to;
    // ...
}
```
## Impact
Core invariant is broken: Admins can influence the odds of a raffle
## Code Snippet
https://github.com/sherlock-audit/2024-08-winnables-raffles/blob/e8b0603f6a155c7505dacc77194ae6789d0dbe7a/public-contracts/contracts/WinnablesTicketManager.sol#L475
https://github.com/sherlock-audit/2024-08-winnables-raffles/blob/e8b0603f6a155c7505dacc77194ae6789d0dbe7a/public-contracts/contracts/WinnablesTicket.sol#L182-L199
## Tool used

Manual Review

## Recommendation
consider restricting the `WinnablesTicket::mint` function so that it can only be called by the `WinnablesTicketManager`.

