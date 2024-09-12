# H-1: Players have complete freedom to customize the fighter NFT when calling `redeemMintPass` and can redeem fighters of types Dendroid and with rare attributes
The function [redeemMintPass](https://github.com/code-423n4/2024-02-ai-arena/blob/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/FighterFarm.sol#L233-L262) allows burning multiple mint passes in exchange for fighters' NFTs. It is mentioned by the sponsor that the player should not have a choice of customizing the fighters' properties and their type. However, nothing prevents a player from:
1. providing `uint8[] fighterTypes` of values `1` to mint fighters of types *Dendroid*. 
2. checking previous transactions in which the [`dna` provided](https://github.com/code-423n4/2024-02-ai-arena/blob/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/FighterFarm.sol#L237) led to minting fighters with rare physical attributes, copying those Dnas and passing them to the [redeemMinPass](https://github.com/code-423n4/2024-02-ai-arena/blob/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/FighterFarm.sol#L237) to mint fighters with low rarity attributes. That is because creating physical attributes is [deterministic](https://github.com/code-423n4/2024-02-ai-arena/blob/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/AiArenaHelper.sol#L83-L121), so providing the same inputs leads to generating a fighter with the same attributes.
## Impact
This issue has two major impacts:
- Players with valid mint passes can mint fighters of type Dendroid easily.
- Players with valid mint passes can mint easily fighters with low rarity attributes which breaks the pseudo-randomness attributes generation aspect

## Proof of Concept
For someone having valid mint passes, he calls the function [redeemMintPass](https://github.com/code-423n4/2024-02-ai-arena/blob/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/FighterFarm.sol#L233) providing [`fighterTypes` array](https://github.com/code-423n4/2024-02-ai-arena/blob/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/FighterFarm.sol#L235) of values *1*. For each mint pass, the inner function [_createNewFighter](https://github.com/code-423n4/2024-02-ai-arena/blob/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/FighterFarm.sol#L257) will be called passing the value *1* as `fighterType` argument which corresponds to *Dendroid*, a new fighter of type [dendroid](https://github.com/code-423n4/2024-02-ai-arena/blob/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/FighterFarm.sol#L509) will be minted for the caller.
```js
function test_redeeming_dendroid_fighters_easily() public {
    uint8[2] memory numToMint = [1, 0];
    bytes memory signature = abi.encodePacked(
        hex"20d5c3e5c6b1457ee95bb5ba0cbf35d70789bad27d94902c67ec738d18f665d84e316edf9b23c154054c7824bba508230449ee98970d7c8b25cc07f3918369481c"
    );
    string[] memory _tokenURIs = new string[](1);
    _tokenURIs[0] = "ipfs://bafybeiaatcgqvzvz3wrjiqmz2ivcu2c5sqxgipv5w2hzy4pdlw7hfox42m";

    // first i have to mint an nft from the mintpass contract
    assertEq(_mintPassContract.mintingPaused(), false);
    _mintPassContract.claimMintPass(numToMint, signature, _tokenURIs);
    assertEq(_mintPassContract.balanceOf(_ownerAddress), 1);
    assertEq(_mintPassContract.ownerOf(1), _ownerAddress);

    // once owning one i can then redeem it for a fighter
    uint256[] memory _mintpassIdsToBurn = new uint256[](1);
    string[] memory _mintPassDNAs = new string[](1);
    uint8[] memory _fighterTypes = new uint8[](1);
    uint8[] memory _iconsTypes = new uint8[](1);
    string[] memory _neuralNetHashes = new string[](1);
    string[] memory _modelTypes = new string[](1);

    _mintpassIdsToBurn[0] = 1;
    _mintPassDNAs[0] = "dna";
    _fighterTypes[0] = 1; // @audit Notice that I can provide value 1 which corresponds to Dendroid type
    _neuralNetHashes[0] = "neuralnethash";
    _modelTypes[0] = "original";
    _iconsTypes[0] = 1;

    // approve the fighterfarm contract to burn the mintpass
    _mintPassContract.approve(address(_fighterFarmContract), 1);

    _fighterFarmContract.redeemMintPass(
    _mintpassIdsToBurn, _fighterTypes, _iconsTypes, _mintPassDNAs, _neuralNetHashes, _modelTypes
    );

    // check balance to see if we successfully redeemed the mintpass for a fighter
    assertEq(_fighterFarmContract.balanceOf(_ownerAddress), 1);
}
```
```bash
Ran 1 test for test/FighterFarm.t.sol:FighterFarmTest
[PASS] test_redeeming_dendroid_fighters_easily() (gas: 578678)
Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 6.56ms

Ran 1 test suite: 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

The player can also inspect previous transactions that minted a fighter with rare attributes, copy the provided `mintPassDnas` and provide them as [argument in the `redeemMintPass`](https://github.com/code-423n4/2024-02-ai-arena/blob/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/FighterFarm.sol#L237). The `_createNewFighter` function [calls `AiArenaHelper`](https://github.com/code-423n4/2024-02-ai-arena/blob/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/FighterFarm.sol#L510) to create the physical attributes for the fighter. The probability of attributes is [deterministic](https://github.com/code-423n4/2024-02-ai-arena/blob/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/AiArenaHelper.sol#L107-L109) and since the player provided `dna` that already led to a fighter with rare attributes, his fighter will also have rare attributes.

## Tools Used
Manual Review

## Recommended Mitigation Steps
The main issue is that the mint pass token is not tied to the fighter properties that the player should claim and the player has complete freedom of the inputs. Consider implementing a signature mechanism that prevents the player from changing the fighter's properties like implemented in [claimFighters](https://github.com/code-423n4/2024-02-ai-arena/blob/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/FighterFarm.sol#L206)


# H-2: A player can stake a small amount of NRN to not have NRN at risk when losing battles due to rounding issue
After each battle, [updateBattleRecord](https://github.com/code-423n4/2024-02-ai-arena/blob/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/RankedBattle.sol#L322) is called by the server to update battle records on-chain for a given fighter. In case of staking NRN or having some NRN at risk, [_addResultPoints](https://github.com/code-423n4/2024-02-ai-arena/blob/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/RankedBattle.sol#L342-L344) will be called, and it will calculate the potential amount of NRNs to put at risk in case of losing the battle or to retrieve from the stake-at-risk in case of winning in [this line](https://github.com/code-423n4/2024-02-ai-arena/blob/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/RankedBattle.sol#L439). The instruction `curStakeAtRisk = (bpsLostPerLoss * (amountStaked[tokenId] + stakeAtRisk)) / 10**4;` is susceptible to rounding issues if the amount staked is small. In a situation where the amount staked is too small, the `currStakeAtRisk` calculation will be rounded down to zero. So even if a fighter loses a battle, [0 NRNs will be transferred to the StakeAtRisk contract](https://github.com/code-423n4/2024-02-ai-arena/blob/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/RankedBattle.sol#L493).
## Impact
- This issue allows someone staking a very small amount of NRN to not have any NRN at risk in case of losing battles, but still earn points in case of winning, as points are determined by the `stakingFactor`(which is [rounded up to 1](https://github.com/code-423n4/2024-02-ai-arena/blob/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/RankedBattle.sol#L530-L532) if it is 0) multiplied by `eloFactor` as [shown here](https://github.com/code-423n4/2024-02-ai-arena/blob/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/RankedBattle.sol#L445).
- This issue violates the purpose of having NRNs at risk that makes the battles valuable
## Proof of Concept
Consider the following scenario:
- A fighter stakes a very small amount of NRN, say for example `30 NRN`
- The fighter loses a battle, and the execution of `_addResultPoints` comes to [this line](https://github.com/code-423n4/2024-02-ai-arena/blob/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/RankedBattle.sol#L439) attempting to calculate the amount of NRNs to put at risk. However, `curStakeAtRisk` will be rounded to zero: `curStakeAtRisk = (10 * (30 + 0)) / 10 ** 4 = 0`. As a result, **[0 amounts of NRN will be transferred](https://github.com/code-423n4/2024-02-ai-arena/blob/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/RankedBattle.sol#L493) to the `StakeAtRisk` contract**
- If the fighter wins a battle, he is still able to earn points and he can earn a good amount of points if his ELO is high
```js
function test_staking_little_amount_causes_to_not_having_a_risk_of_having_NRN_tokens_at_risk_but_earning_good_points() public {
    address player = vm.addr(3);
    _mintFromMergingPool(player);
    uint8 tokenId = 0;
    _fundUserWith4kNeuronByTreasury(player);
    vm.prank(player);
    // staking a small amount of $NRN 
    _rankedBattleContract.stakeNRN(30, 0);
    assertEq(_rankedBattleContract.amountStaked(0),  30);

    // The battle ends, the fighter lost the battle
    vm.prank(address(_GAME_SERVER_ADDRESS));
    _rankedBattleContract.updateBattleRecord(tokenId, 50, 2, 3500, true);

    uint256 nrnAtRisk = _stakeAtRiskContract.getStakeAtRisk(tokenId);
    assertEq(0, nrnAtRisk); // @audit : 0 NRN at risk even when losing battles
}
```
```bash
Ran 1 test for test/RankedBattle.t.sol:RankedBattleTest
[PASS] test_staking_little_amount_causes_to_not_having_a_risk_of_having_NRN_tokens_at_risk_but_earning_good_points() (gas: 776640)
Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 2.77ms

Ran 1 test suite: 1 tests passed, 0 failed, 0 skipped (1 total tests)
```
## Tools Used
Manual Review

## Recommended Mitigation Steps
The main reason for this issue is that a user can stake a very small amount of NRN causing some mathematical calculations to round to zero, consider checking for a minimum value of NRN to stake.