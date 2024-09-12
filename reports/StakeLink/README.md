# A potential state disaster and ownerships loss of locks on `SDLPoolSecondary` if lock updates are not dispatched in the correct order by the user

## Relevant GitHub Links
https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/sdlPool/SDLPoolSecondary.sol#L428C14-L428C30
https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/sdlPool/SDLPoolSecondary.sol#L472

## Summary 

In the secondary pool, a user can mistakenly update the same lockId he already fully withdrew (and the full withdrawal is on the queue) by staking more on it. Once the updates are executed, the staked amount will be permanently locked in the `SDLPoolSecondary` contract without the possibility of withdrawal because the ownership of the lockId is removed during the execution of the first full withdrawal update.

## Vulnerability Details
On the Secondary Pool, when a user performs any action that will alter their effective reSDL balance such as staking, locking, withdrawing, or initiating a withdrawal, the action will be queued instead of taking effect immediately. A user can perform as many updates as he wants on a given lock during a batch. Once an update comes from the primary pool, the queued updates will be executed in their requested order, **and if a lock has several updates that are not dispatched in the correct order, a state disaster can occur**.

Consider the following series of updates on a given `lockId 1` that belongs to Bob in which after the execution of updates there will be a permanent lock of SDLToken on the Secondary Pool contract with ownership loss of the lock:

- Bob is the owner of a lockId 1 with the following information: `Lock(amount:100, boostAmount:0, startTime: 0, duration: 0, expiry: 0)`
- Bob withdraws all his staked amount in lockId 1, the action will be queued and pushed to `queuedLockUpdates`
- Bob stakes on the same lockId 1 by transferring 200 SDL tokens, so `onTokenTransfer` is called and the `_calldata` passed will be `(1, 0)` (lockId 1 with 0 lockingDuration), the update is queued by pushing it to `queuedLockUpdates`

Suppose now the `CCIPController` sends the update to the primary chain and the updates are confirmed:

- Bob calls the function `executeQueuedOperations` to execute his pending operations
- The internal function `_executeQueuedLockUpdates` will be executed, it will loop through the pending updates of the lockId 1:
  * The first update is a full withdrawal of the staked amount in lockId 1, so [this block](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/sdlPool/SDLPoolSecondary.sol#L470) will be executed. **Notice that** **`delete lockOwners[lockId];`** **and** **`balances[_owner] -= 1;`** will be executed and the ownership is removed from lockId 1
  * The second queued update gets executed in the second iteration of the loop, and as the [baseAmountDiff](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/sdlPool/SDLPoolSecondary.sol#L465) is greater than 0, [this block](https://github.com/Cyfrin/2023-12-stake-link/blob/549b2b8c4a5b841686fceb9c311dca9ac58225df/contracts/core/sdlPool/SDLPoolSecondary.sol#L484) will be executed. The `locks[lockId]` gets updated to the new lock state, the `effectiveBalance` of Bob gets increased by 200, and the `totalEffectiveBalance` gets increased by 200 **BUT the** **`lockOwners[1]`** **is still deleted in the first update and is not handled in the second update**

After the execution of updates, `locks[1]` will contain valid stake information (`locks[1] = Lock(amount:200, boostAmount:0, startTime: 0, duration: 0, expiry: 0)`) but with no owner associated to it as `lockOwners[1]` is deleted in the first update, so the stakes are locked in the contract and Bob can not withdraw his stakes as he is no longer the owner of lockId 1. The pool has no mechanism to recover the permanent locked SDL.

## PoC

```js
it("Stakes can be locked in the contract if there's a queued update on a lockId after a full withdraw queue", async () => {
    // Bob is the owner of lockId1 with 100 stake amount
    await mintLock(false)
    
    // Bob withdraws the full staked 100 SDL on the lockId1 he owns. This operation is on the queue
    await sdlPool.withdraw(1, toEther(100))
    // Bob mistakenly stakes more on the lockId1 by sending 200 SDL tokens. This operation is on the queue
    await sdlToken.transferAndCall(
      sdlPool.address,
      toEther(200),
      ethers.utils.defaultAbiCoder.encode(['uint256', 'uint64'], [1, 0])
    )
    // queued updates are sent to the primary chain and get confirmed
    await updateLocks()

    // The 2 queued updates get executed
    assert.equal(fromEther(await sdlToken.balanceOf(sdlPool.address)), 200) // relative to second update
    assert.equal(fromEther(await sdlPool.totalEffectiveBalance()), 200) // relative to second update
    assert.equal(fromEther(await sdlPool.totalStaked()), 200) // relative to second update
    assert.equal(fromEther(await sdlPool.effectiveBalanceOf(accounts[0])), 200) // relative to second update
    assert.equal(fromEther(await sdlPool.staked(accounts[0])), 200) // relative to second update

    // Bob has 0 balance because `balances[_owner] -= 1;` got executed on the first full withdrawal update
    // but not updated on the second update
    assert.equal((await sdlPool.balanceOf(accounts[0])).toNumber(), 0)
    // Bob is no longer the owner of the stake. The 200 SDL tokens are permanently locked
    await expect(sdlPool.ownerOf(1)).to.be.revertedWith('InvalidLockId()')
  })
```

## Impact
This issue will have the following impacts:

- Permanent lock of stakes on the secondary pool without a possiblity of recovery.
- Users may lose control on their stakes

## Tools Used
Manual Review

## Recommendations
Consider preventing users from updating a lock if there's already a full withdrawal of the lock on the queue. Below is a suggestion code to fix the issue:
```solidity
function _queueLockUpdate(
        address _owner,
        uint256 _lockId,
        uint256 _amount,
        uint64 _lockingDuration
    ) internal onlyLockOwner(_lockId, _owner) {
    Lock memory lock = _getQueuedLockState(_lockId);
    if (lock.amount == 0) revert AlreadyFullWithdrawed();
    ...
```
