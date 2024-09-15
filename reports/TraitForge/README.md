# H-01 Generations can be incremented beyond the maximum allowed
## Impact
Each generation has a predefined cap on the number of tokens that can be minted. Once this cap is reached, the [_incrementGeneration](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/TraitForgeNft/TraitForgeNft.sol#L345-L355) function is triggered to advance the game to the next generation. The issue is that **the function does not check if the new generation exceeds the maximum generation allowed**:
```solidity
function _incrementGeneration() private {
    require(
      generationMintCounts[currentGeneration] >= maxTokensPerGen,
      'Generation limit not yet reached'
    );
    currentGeneration++;
    generationMintCounts[currentGeneration] = 0; /
    priceIncrement = priceIncrement + priceIncrementByGen;
    entropyGenerator.initializeAlphaIndices();

    // @audit No check if the new generation exceeds the maximum generation allowed
    emit GenerationIncremented(currentGeneration);
}
```
As shown above, the generation is incremented without checking if it exceeds the maximum generation allowed. This allows minting tokens on generations beyond `maxGeneration`:
```solidity
function _mintInternal(address to, uint256 mintPrice) internal {
    if (generationMintCounts[currentGeneration] >= maxTokensPerGen) {
      _incrementGeneration();
    }

    _tokenIds++;
    uint256 newItemId = _tokenIds;
    _mint(to, newItemId);
    uint256 entropyValue = entropyGenerator.getNextEntropy();

    tokenCreationTimestamps[newItemId] = block.timestamp;
    tokenEntropy[newItemId] = entropyValue;
    tokenGenerations[newItemId] = currentGeneration;
    generationMintCounts[currentGeneration]++;
    initialOwners[newItemId] = to;

    if (!airdropContract.airdropStarted()) {
      airdropContract.addUserAmount(to, entropyValue);
    }

    emit Minted(
      msg.sender,
      newItemId,
      currentGeneration,
      entropyValue,
      mintPrice
    );

    _distributeFunds(mintPrice);
  }
```

## Proof of Concept
Copy/Paste the following test in `test/TraitForgeNft.test.ts`
```ts
it('No check for max generation', async () => {
  await nft.setMaxGeneration(1);
  await nft.setMaxTokensPerGen(3); // We added this function in the smart contract for test simplicity
  await entropyGenerator.transferOwnership(await nft.getAddress()) // we need to transfer ownership so _incrementGeneration does not revert

  // mint token 1 to user 1 => gen(token1) = 1
  await nft.connect(user1).mintToken(merkleInfo.whitelist[1].proof, {
    value: ethers.parseEther('1'),
  });

  // mint token 2 to user 1 => gen(token2) = 1
  await nft.connect(user1).mintToken(merkleInfo.whitelist[1].proof, {
    value: ethers.parseEther('1'),
  });

  // mint token 3 to user 2 => gen(token3) = 1 
  await nft.connect(user2).mintToken(merkleInfo.whitelist[2].proof, {
    value: ethers.parseEther('1'),
  });


  // user2 mints a new token. Since the generation 1 reached the tokens limit per generation (3) 
  // and the maxGen is 1, the following transaction should revert, but it will not

  await nft.connect(user2).mintToken(merkleInfo.whitelist[2].proof, {
    value: ethers.parseEther('1'),
  });

  const token4Generation = await nft.tokenGenerations(4);
  assert.equal(2, Number(token4Generation)) // Generation is increased to 2, beyond the maximum allowed
})
```
```bash
yarn test --grep "No check for max generation"
```
**NOTE1**: `entropyGenerator` is needed in the test to transfer the ownership of the `entropyGenerator` to `nft` so the `_incrementGeneration` will [not revert](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/TraitForgeNft/TraitForgeNft.sol#L353), and `entropyGenerator` is not set globally form the base describe block, so make sure to set it as global variable:
```diff
describe('TraitForgeNFT', () => {
  let entityForging: EntityForging;
  let nft: TraitForgeNft;
+  let entropyGenerator: EntropyGenerator;
  let owner: HardhatEthersSigner;

  //...
-  const entropyGenerator = (await EntropyGenerator.deploy(await nft.getAddress())) as EntropyGenerator;
+   entropyGenerator = (await EntropyGenerator.deploy(await nft.getAddress())) as EntropyGenerator;
  //...
}
```
**Note2**: For simplicity purposes, we added a new function in `TraitForgeNft` to set the `MaxTokensPerGen` manually (so we don't need to mint 10,000 tokens):
```solidity
function setMaxTokensPerGen(uint _max) external onlyOwner {
    maxTokensPerGen = _max;
}
```
## Tools Used
Manual Review
## Recommended Mitigation Steps
Consider checking if the new incremented generation does not exceed the maximum allowed. Below is a suggested update to the `TraitForgeNft::_incrementGeneration` function:
```diff
function _incrementGeneration() private {
    require(
      generationMintCounts[currentGeneration] >= maxTokensPerGen,
      'Generation limit not yet reached'
    );
  currentGeneration++;
+  require(currentGeneration <= maxGeneration, "Current generation exceeds the maximum allowed");
  generationMintCounts[currentGeneration] = 0; /
  priceIncrement = priceIncrement + priceIncrementByGen;
  entropyGenerator.initializeAlphaIndices();

    emit GenerationIncremented(currentGeneration);
}
```

# H-02 `_incrementGeneration` resets `generationMintCounts` and ignores the tokens already forged on the new generation

## Impact
Each generation has a predefined cap on the number of tokens that can be minted. Once this cap is reached, the [_incrementGeneration](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/TraitForgeNft/TraitForgeNft.sol#L345-L355) function is triggered to advance the game to the next generation. The issue is that the function sets the `generationMintCounts` of the new generation to zero, **ignoring the tokens that were already forged into the new generation**:
```solidity
function _incrementGeneration() private {
    require(
      generationMintCounts[currentGeneration] >= maxTokensPerGen,
      'Generation limit not yet reached'
    );
    currentGeneration++;
    generationMintCounts[currentGeneration] = 0; // @audit Wrong update
    priceIncrement = priceIncrement + priceIncrementByGen;
    entropyGenerator.initializeAlphaIndices();
    emit GenerationIncremented(currentGeneration);
}
```
As shown above, the mint count for the new generation is *reset* to zero `generationMintCounts[currentGeneration] = 0;`. When forging two tokens, the new token will always be of a [new generation](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/TraitForgeNft/TraitForgeNft.sol#L163):
```solidity
function forge(
    address newOwner,
    uint256 parent1Id,
    uint256 parent2Id,
    string memory
  ) external whenNotPaused nonReentrant returns (uint256) {
    require(
      msg.sender == address(entityForgingContract),
      'unauthorized caller'
    );
    uint256 newGeneration = getTokenGeneration(parent1Id) + 1;

    /// Check new generation is not over maxGeneration
    require(newGeneration <= maxGeneration, "can't be over max generation");

    // ...
}
```
Therefore, if tokens were already forged into the new generation, the mint counts will be ignored when incrementing the current generation and will be set to zero, resulting in `generationMintCounts` containing an *invalid* state. This has two impacts:
- It allows for minting more tokens than intended for the generation, exceeding the generation cap
- `calculatePrice` will return unintended lower values because it depends on `generationMintCounts`
## Proof Of Concept
Consider the following example demonstrated by runnable PoC below:
- The maximum number of tokens per generation is set to `3`.
- `User1` mints `2` tokens and `User2` mints `1` token. All three tokens belong to generation `1`.
- `User1` lists his token 1 for forging and `User2` forges the listed token. 
    - The new token, with ID 4, belongs to generation `2` since forging new tokens always results in a [new generation](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/TraitForgeNft/TraitForgeNft.sol#L163):
    ```solidity
    // TraitForgeNft::forge
    uint256 newGeneration = getTokenGeneration(parent1Id) + 1;
    ```
- At this point, `generationMintCounts[1] = 3` and `generationMintCounts[2] = 1`
- Now, `User1` mints a new token (of id 5). Since the current generation `1` has reached the maximum tokens per generation (3), the current generation will increase:
    ```solidity
    // TraitForgeNft::_mintInternal

    function _mintInternal(address to, uint256 mintPrice) internal {
        if (generationMintCounts[currentGeneration] >= maxTokensPerGen) {
        _incrementGeneration();
        }
        // ...
    }
    ```
    - The `_incrementGeneration` will set the `generationMintCounts[2]` to `0`, ignoring the previously forged token into generation `2`:
    ```solidity
    function _incrementGeneration() private {
        //...
        currentGeneration++;
        generationMintCounts[currentGeneration] = 0;    
        //...
    }
    ```

To demonstrate this, copy and paste the following test to  `test/TraitForgeNft.test.ts`:
```ts
describe("PoC", () => {
    it("_incrementGeneration incorrectly sets the mints for new generation to zero", async () => {
      await entropyGenerator.transferOwnership(await nft.getAddress())
      await nft.setMaxTokensPerGen(3); // We added this function in the smart contract for test simplicity

      // mint token 1 to user 1 => gen(token1) = 1
      await nft.connect(user1).mintToken(merkleInfo.whitelist[1].proof, {
        value: ethers.parseEther('1'),
      });

      // mint token 2 to user 1 => gen(token2) = 1
      await nft.connect(user1).mintToken(merkleInfo.whitelist[1].proof, {
        value: ethers.parseEther('1'),
      });

      // mint token 3 to user 2 => gen(token3) = 1 
      await nft.connect(user2).mintToken(merkleInfo.whitelist[2].proof, {
        value: ethers.parseEther('1'),
      });

      // user1 lists his token 1 for forge
      await entityForging.connect(user1).listForForging(1, ethers.parseEther('0.01'))
      // user2 forges the listed token 1
      await expect(entityForging.connect(user2).forgeWithListed(1, 3, {value: ethers.parseEther('0.1')}))
        .to.emit(entityForging, "EntityForged")

        // The new minted token because of forge will go to new generation (2)
      const token4Generation = await nft.getTokenGeneration(4)
      assert.equal(2, Number(token4Generation))
      
      // Ensure that mint counts for generation 2 is 1 at this moment (because of token 4)
      let generation2MintCounts = await nft.generationMintCounts(2)
      assert.equal(1, Number(generation2MintCounts))

      // user2 will mint new token, because gen1 count reached the limit, his new token will be on generation 2
      await nft.connect(user2).mintToken(merkleInfo.whitelist[2].proof, {
        value: ethers.parseEther('1'),
      });

      const token5Generation = await nft.getTokenGeneration(5)
      assert.equal(2, Number(token5Generation))

      // The mint count for generation 2 will be 1 instead of 2
      generation2MintCounts = await nft.generationMintCounts(2)
      assert.equal(1, Number(generation2MintCounts))
    })
})
```
```bash
yarn test --grep "_incrementGeneration incorrectly sets the mints for new generation to zero"
```

**NOTE1**: `entropyGenerator` is needed in the test to transfer the ownership of the `entropyGenerator` to `nft` so the `_incrementGeneration` will [not revert](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/TraitForgeNft/TraitForgeNft.sol#L353), and `entropyGenerator` is not set globally form the base describe block, so make sure to set it as global variable:
```diff
describe('TraitForgeNFT', () => {
  let entityForging: EntityForging;
  let nft: TraitForgeNft;
+  let entropyGenerator: EntropyGenerator;
  let owner: HardhatEthersSigner;

  //...
-  const entropyGenerator = (await EntropyGenerator.deploy(await nft.getAddress())) as EntropyGenerator;
+   entropyGenerator = (await EntropyGenerator.deploy(await nft.getAddress())) as EntropyGenerator;
  //...
}
```
**Note2**: For simplicity purposes, we added a new function in `TraitForgeNft` to set the `MaxTokensPerGen` manually (so we don't need to mint 10,000 tokens):
```solidity
function setMaxTokensPerGen(uint _max) external onlyOwner {
    maxTokensPerGen = _max;
}
```

## Tools used
Manual Review

## Recommended Mitigation Steps
Ensure that the tokens already forged into the new generation are considered when updating  `generationMintCounts` in `TraitForgeNft::_incrementNewGeneration`

# H-03 Missing `pause/unpause` functionalities in contracts inheriting pausable
## Impact
The following contracts in scope: `DevFund`, `EntityForging`, `EntityTrading`, `EntropyGenerator`, `NukeFund`, `TraitForgeNft` are supposed to be pausable (as they all inherit from `Pausable`), but they don't implement the external pause/unpause functionalities which means it will never be possible to pause them.

The OpenZeppelin `Pausable` contract contains internal functions (`_pause` and `_unpause`) to manage the paused state of the contract. Any contract inheriting Pausable **MUST** implement external functions that call these internal functions to enable pausing and unpausing.

## Tools used
Manual Review

## Recommended Mitigation Steps
Add public/external pause and unpause functions in the mentioned contracts to enable their pausable functionality. This can be done as shown in the example below:
```solidity
function pause() external onlyOwner {
  _pause();
}

function unpause() external onlyOwner {
  _unpause();
}
```

# H-04 Incorrect Usage of `taxCut` for Fee Calculation
## Impact
The `taxCut` parameter is used in [EntityForging](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/EntityForging/EntityForging.sol#L146), [EntityTrading](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/EntityTrading/EntityTrading.sol#L72), and [NukeFund](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/NukeFund/NukeFund.sol#L41) to calculate fees distributed to various ecosystem actors, such as developer fees and NukeFund fees. For example, in `EntityForging`:
```solidity
// EntityForgin::forgeWithListed

uint256 devFee = forgingFee / taxCut;
```
The issue is that the fees are incorrectly calculated by dividing the amount by `taxCut`. This calculation is accurate when `taxCut` is set to `10` (representing 10%), but the `taxCut` can change frequently through the `setTaxCut` function. If the `taxCut` is set to a value different from `10`, the fees will be incorrectly calculated. For instance, if the taxCut is set to `2` (representing 2%), the `devFee` above will actually be `50%` instead of `2%`.

## Tools Used
Manual Review
## Recommended Mitigation Steps
Consider updating the fee calculation formula as follows:
```solidity
uint256 devFee = (forgingFee * taxCut) / 100; // EntityForgin::forgeWithListed
uint256 devShare = (msg.value * taxCut) / 100; // NukeFund::receive()
uint256 nukeFundContribution = (msg.value * taxCut) / 100; // EntityTrading::buyNFT
```

# H-05 `getEntropy` Fails to Handle Small Slot Values Leading to Incorrect Entropy Extraction
## Impact
Entropy values are stored in uint256 slots, with each slot containing 13 concatenated entropies. The function `getEntropy` returns the entropy value based on the slot and number index:
```solidity
function getEntropy(
    uint256 slotIndex,
    uint256 numberIndex
  ) private view returns (uint256) {
    require(slotIndex <= maxSlotIndex, 'Slot index out of bounds.');

    if (
      slotIndex == slotIndexSelectionPoint &&
      numberIndex == numberIndexSelectionPoint
    ) {
      return 999999;
    }

    uint256 position = numberIndex * 6; // calculate the position for slicing the entropy value
    require(position <= 72, 'Position calculation error');

    uint256 slotValue = entropySlots[slotIndex]; // slice the required art of the entropy value
    uint256 entropy = (slotValue / (10 ** (72 - position))) % 1000000; // adjust the entropy value based on the number of digits
    uint256 paddedEntropy = entropy * (10 ** (6 - numberOfDigits(entropy)));

    return paddedEntropy; // return the caculated entropy value
}
```
The entropy value is extracted from the given slot at a specific position using the following formula:
```solidity
uint256 entropy = (slotValue / (10 ** (72 - position))) % 1000000;
```
While the math above is correct, it does not handle the case where the `slotValue` (which contains 13 concatenated entropies) is smaller than `10 ** (72 - position)`. In such cases, the entropy value will be rounded down to zero due to Solidity's division behavior. This issue is more likely to occur when extracting the first entropy values of the slot (most significant 6 digits) because the position would be small (e.g., 0 for the first entropy, and the denominator will be a large number `10**72`). Consider for example this slot value:
![random slot value](https://i.imgur.com/wboQs7b.png)
- Attempting to extract the first entropy would resolve in this calculation: `(slotValue/10**72) % 1000000` which will be rounded to zero because `slotValue < 10**72`, which is wrong as it should be `812001`
- Attempting to extract the second entropy would resolve in this calculation: `(slotValue/10**66) % 1000000` which will be rounded to zero because `slotValue < 10**66`, which is wrong as it should be `005699`


It is important to note that a `slotValue` smaller than the denominator (especially when extracting the first entropies) is not negligible because its value is [pseudo-randomly calculated](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/EntropyGenerator/EntropyGenerator.sol#L73-L77) in the batch writing process:
```solidity
uint256 pseudoRandomValue = uint256(keccak256(abi.encodePacked(block.number, i))) % uint256(10) ** 78;
```
The slot value ranges between `0` and `(10**78) - 1`, making it non-negligible that a slot value is smaller than `10 ** (72 - position)`, especially when extracting the first entropies as `position` will be a small value.
## Tools Used
Manual Review
## Recommendation Mitigation Steps


# H-05 Forgers can forge one additional time beyond their forge potential for every token
## Impact
The `listForForging` function in the `EntityForging` contract allows the owner of a token, specifically a forger entity, to list it for forging. It [checks](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/EntityForging/EntityForging.sol#L88) if the forging counts did not exceed the forge potential. However, the check is incorrect:
```solidity
function listForForging(
  uint256 tokenId,
  uint256 fee
) public whenNotPaused nonReentrant {
  // ...
  uint256 entropy = nftContract.getTokenEntropy(tokenId); // Retrieve entropy for tokenId
  uint8 forgePotential = uint8((entropy / 10) % 10); // Extract the 5th digit from the entropy
  require(
    forgePotential > 0 && forgingCounts[tokenId] <= forgePotential, // @audit Wrong check
    'Entity has reached its forging limit'
  );

  // ...
}
```
The check `forgingCounts[tokenId] <= forgePotential` is incorrect because it allows the forger to forge one additional time beyond the forge potential. This happens because the `forgingCounts[tokenId]` is incremented only after the  [the merge occurs in `forgeWithListed` function](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/EntityForging/EntityForging.sol#L133). Therefore, the check does not account for the current forging transaction. This can occur for every token held by the forger.

## Proof Of Concept
Consider the following example:
- Entity `A` with the forger role has a forge potential of `1` and owns tokenId `1`.
- Entity `A` lists its token for forging by calling `listForForging`. Notice that `forgingCounts[1]` is NOT incremented at this stage.
- Entity `B` with the merger role calls `forgeWithListed` to merge the previously listed token. During this transaction, `forgingCounts[1]` [is incremented]((https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/EntityForging/EntityForging.sol#L133)). At this point, entity `A` should have consumed all its forge potential.
- Entity `A` attempts to list its tokenId `1` for forging again by calling `listForForging`. The check that verifies if entity `A` has consumed its forge potential will pass, even though it should not:
  ```solidity
  require(forgePotential > 0 && forgingCounts[tokenId] <= forgePotential, 'Entity has reached its forging limit');
  ```
  This is because `forgingCounts[1]` equals `1`, and `forgePotential` equals to `1`, so `1<=1`.
- Entity `B` with the merger role calls `forgeWithListed` to merge the previously listed token.

Entity `A` could forge its token id one additional time beyond its forge potential.
## Tools Used
Manual Review

## Recommended Mitigation Steps
Consider updating the previous check to the following:
```diff
 require(forgePotential > 0 && 
- forgingCounts[tokenId] <= forgePotential
+ forgingCounts[tokenId] < forgePotential
 , 'Entity has reached its forging limit');
```

# M-01 Minting tokens will be impossible when `generationMintCounts` cap is reached for the first generation

## Impact
Each generation has a predefined cap on the number of tokens that can be minted. Once this cap is reached, the [_incrementGeneration](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/TraitForgeNft/TraitForgeNft.sol#L345-L355) function is triggered to advance the game to the next generation:
```solidity
function _incrementGeneration() private {
    require(
      generationMintCounts[currentGeneration] >= maxTokensPerGen,
      'Generation limit not yet reached'
    );
    currentGeneration++;
    generationMintCounts[currentGeneration] = 0;
    priceIncrement = priceIncrement + priceIncrementByGen;
    entropyGenerator.initializeAlphaIndices(); // @audit call to restricted function on `EntropyGenerator` contract
    emit GenerationIncremented(currentGeneration);
}
```
The issue is that the function calls `EntropyGenerator::initializeAlphaIndices` which is a [restricted function](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/EntropyGenerator/EntropyGenerator.sol#L206C58-L206C67) limited to the conract owner:
```solidity
// EntropyGenerator

function initializeAlphaIndices() public whenNotPaused onlyOwner {//...);
```
The intended owner for the `EntropyGenerator` is NOT the `TraitForgeNft` contract, but a multisig contract held by the team as confirmed by the sponsor. Even if the team transfers the ownership of `EntropyGenerator` to `TraitForgeNft` to prevent the DoS, **there is no way to return back the ownership to the multisig account**. Therefore a damage **will always occure** regardless of the team's decision:
1. If the team transfers the ownership of `EntropyGenerator` to `TraitForgeNft` to enable further minting and prevent DoS, there is no way to return the ownership to the multisig account.
2. If the `EntropyGenerator`'s owner is intended to always be the multisig held by the team, a DoS will occur once the maximum tokens per generation are reached for the first generation, as the current generation cannot be increased anymore.
## Tools Used
Manual Review

## Recommended Mitigation Steps
Since `EntropyGenerator::initializeAlphaIndices` is called either at deployment or by the `TraitForgeNft` contract to increase the generation, consider restricting the function to `onlyAllowedCaller` instead of `onlyOwner`. Below is a suggested code update:
```diff
// EntropyGenerator contract


constructor(address _traitForgetNft) {
  allowedCaller = _traitForgetNft;
-  initializeAlphaIndices();
+  _initializeAlphaIndices();
}

+function initializeAlphaIndices() public whenNotPaused onlyAllowedCaller {
+ _initializeAlphaIndices();
+}

-function initializeAlphaIndices() public whenNotPaused onlyOwner {
+function _initializeAlphaIndices() internal {

    uint256 hashValue = uint256(
      keccak256(abi.encodePacked(blockhash(block.number - 1), block.timestamp))
    );

    uint256 slotIndexSelection = (hashValue % 258) + 512;
    uint256 numberIndexSelection = hashValue % 13;

    slotIndexSelectionPoint = slotIndexSelection;
    numberIndexSelectionPoint = numberIndexSelection;
}
```

# M-02 Incorrect check against golden entropy value in the first two batches
## Impact
Entropy is stored in `uint256` slots, with each slot able to contain 13 concatenated entropies. Entropies are generated in three passes, and there is a *golden entropy* value `999999` that should not be generated in the first two passes, [as specified in the documentation](https://github.com/TraitForge/GitBook/blob/main/GamePlay/Entropy.md#entropy):
> There is a certain entropy, “999999” which is referred to as “the Golden God”, since it has perfect parameters and will exceed all other entities if played correctly. **The Golden God is scanned for and is kept out of the first 2 passes**, but is deliberately set in the final pass (at some random point)

The issue is that [writeEntropyBatch1](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/EntropyGenerator/EntropyGenerator.sol#L56) and [writeEntropyBatch2](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/EntropyGenerator/EntropyGenerator.sol#L76) functions do not correctly check for the golden entropy:
```solidity
function writeEntropyBatch1() public {
    require(lastInitializedIndex < batchSize1, 'Batch 1 already initialized.');

    uint256 endIndex = lastInitializedIndex + batchSize1; // calculate the end index for the batch
    unchecked {
      for (uint256 i = lastInitializedIndex; i < endIndex; i++) {
        uint256 pseudoRandomValue = uint256(
          keccak256(abi.encodePacked(block.number, i))
        ) % uint256(10) ** 78; // generate a  pseudo-random value using block number and index
        require(pseudoRandomValue != 999999, 'Invalid value, retry.'); // @audit INCORRECT check
        entropySlots[i] = pseudoRandomValue; // store the value in the slots array
      }
    }
    lastInitializedIndex = endIndex;
  }
```
The function(same for `writeEntropyBatch2`) attempts to ensure that no golden entropy is generated using the following check: `require(pseudoRandomValue != 999999, 'Invalid value, retry.');`. However, this check is incorrect because `pseudoRandomValue` does NOT represent a *single* entropy value but **represents the entire slot value, which contains 13 entropies**. For example, if the pseudo-randomly generated slot value is `........999999123345`, the check will pass while it should not because the second entropy is a golden one.
## Tools Used
Manual review
## Recommended Mitigation Steps
The current check incorrectly validates the entire slot value. Consider updating the check to validate each of the 13 entropies within the slot value.

# M-03 Incorrect check makes `mintWithBudget` ineffective after first generation minting
## Impact
The `mintWithBudget` function allows players to mint multiple entities in one transaction. The function continues minting new entities as long as the remaining budget is sufficient for the next entity to mint. However, the function contains an incorrect check that renders it unusable after the first generation of entities is minted:
```solidity
function mintWithBudget(
    bytes32[] calldata proof
  )
    public
    payable
    whenNotPaused
    nonReentrant
    onlyWhitelisted(proof, keccak256(abi.encodePacked(msg.sender)))
  {
    uint256 mintPrice = calculateMintPrice();
    uint256 amountMinted = 0;
    uint256 budgetLeft = msg.value;

    while (budgetLeft >= mintPrice && _tokenIds < maxTokensPerGen) { // @audit wrong check
      _mintInternal(msg.sender, mintPrice);
      amountMinted++;
      budgetLeft -= mintPrice;
      mintPrice = calculateMintPrice();
    }
    if (budgetLeft > 0) {
      (bool refundSuccess, ) = msg.sender.call{ value: budgetLeft }('');
      require(refundSuccess, 'Refund failed.');
    }
}
```
The check `_tokenIds < maxTokensPerGen` is incorrect. This check prevents the function from being used after the first generation's entities are minted since `_tokenIds` is incremented after each mint.

## Proof Of Concept
Copy and Paste the following test in `test/TraitForgeNft.test.s`:
```ts
it('mintWithBudget will be useless after first generation', async () => {
  await nft.setMaxTokensPerGen(3); // We added this function in the smart contract for simplicity
  await entropyGenerator.transferOwnership(await nft.getAddress())

  // mint token 1 to user 1 => gen(token1) = 1
  await nft.connect(user1).mintToken(merkleInfo.whitelist[1].proof, {
    value: ethers.parseEther('1'),
   });

  // mint token 2 to user 1 => gen(token2) = 1
  await nft.connect(user1).mintToken(merkleInfo.whitelist[1].proof, {
    value: ethers.parseEther('1'),
  });

  // mint token 3 to user 2 => gen(token3) = 1 
  await nft.connect(user2).mintToken(merkleInfo.whitelist[2].proof, {
    value: ethers.parseEther('1'),
   });

  // at this point, generation1's entities are minted (3)

  const totalSupplyBefore = await nft.totalSupply()
  console.log("Total supply before: ", totalSupplyBefore)

  await nft.connect(user1).mintWithBudget(merkleInfo.whitelist[1].proof, {
    value: ethers.parseEther('10'),
  })
  const totalSupplyAfter = await nft.totalSupply()

  // No token was minted
  assert.equal(totalSupplyAfter, totalSupplyBefore)
})
```
```bash
yarn test --grep "mintWithBudget will be useless after first generation"
```

**NOTE1**: `entropyGenerator` is needed in the test to transfer the ownership of the `entropyGenerator` to `nft` so the `_incrementGeneration` will [not revert](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/TraitForgeNft/TraitForgeNft.sol#L353), and `entropyGenerator` is not set globally form the base describe block, so make sure to set it as global variable:
```diff
describe('TraitForgeNFT', () => {
  let entityForging: EntityForging;
  let nft: TraitForgeNft;
+  let entropyGenerator: EntropyGenerator;
  let owner: HardhatEthersSigner;

  //...
-  const entropyGenerator = (await EntropyGenerator.deploy(await nft.getAddress())) as EntropyGenerator;
+   entropyGenerator = (await EntropyGenerator.deploy(await nft.getAddress())) as EntropyGenerator;
  //...
}
```
**Note2**: For simplicity purposes, we added a new function in `TraitForgeNft` to set the `MaxTokensPerGen` manually (so we don't need to mint 10,000 tokens):
```solidity
function setMaxTokensPerGen(uint _max) external onlyOwner {
    maxTokensPerGen = _max;
}
```
## Tools Used
Manual Review

## Recommended Mitigation Steps
The incorrect check is supposed to prevent players from minting more than the allowed number of tokens per generation in one batch. Update the function as follows:
```diff
 function mintWithBudget(
    bytes32[] calldata proof
  )
    public
    payable
    whenNotPaused
    nonReentrant
    onlyWhitelisted(proof, keccak256(abi.encodePacked(msg.sender)))
  {
    uint256 mintPrice = calculateMintPrice();
    uint256 amountMinted = 0;
    uint256 budgetLeft = msg.value;

    while (budgetLeft >= mintPrice &&
-       _tokenIds < maxTokensPerGen
+       amountMinted < maxTokensPerGen
    ){
      _mintInternal(msg.sender, mintPrice);
      amountMinted++;
      budgetLeft -= mintPrice;
      mintPrice = calculateMintPrice();
    }
    if (budgetLeft > 0) {
      (bool refundSuccess, ) = msg.sender.call{ value: budgetLeft }('');
      require(refundSuccess, 'Refund failed.');
    }
  }
```

# M-04 Lack of slippage protection in `nuke` function
## Impact
Entity holders can claim a part of the `Nuke` fund by *nuking* their entity via [nuke function](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/NukeFund/NukeFund.sol#L153-L182). The amount claimable depends on the nuke factor of the entity and the funds held by the contract. The issue is that the function does not protect users from slippage between the moment the transaction is created and the moment it is executed. As a result, users might not receive the funds they expected for the NFT they burned.

## Proof Of Concept
Consider the following example:
1. `NukeFund`'s balance is `5 ETH`
2. Alice with entity A of `50%` nuke factor initializes transaction to nuke her entity, expecting to receive `2.5 ETH`
3. At the same time, Bob with entity B of `40%` nuke factor initializes transaction to nuke his entity, expecting to receive `2 ETH`.
4. Bob's transaction is executed first and receives `2 ETH` as expected. The contract's fund becomes `3 ETH`. 
5. Alice's transaction is executed after and receives `1.5 ETH`.

Alice burned her NFT expecting to receive `2.5 ETH` but received `1.5 ETH`.

## Tools Used
Manual Review

## Recommended Mitigation Steps
Consider allowing players to introduce a minimum claimable amount and revert the transaction if the claimable funds are below the minimum. Below is a suggestion for updated code for the `nuke` function:
```diff
function nuke(
  uint256 tokenId
+ uint256 minimumClaim
) public whenNotPaused nonReentrant {
    require(
      nftContract.isApprovedOrOwner(msg.sender, tokenId),
      'ERC721: caller is not token owner or approved'
    );
    require(
      nftContract.getApproved(tokenId) == address(this) ||
        nftContract.isApprovedForAll(msg.sender, address(this)),
      'Contract must be approved to transfer the NFT.'
    );
    require(canTokenBeNuked(tokenId), 'Token is not mature yet');

    uint256 finalNukeFactor = calculateNukeFactor(tokenId); // finalNukeFactor has 5 digits
    uint256 potentialClaimAmount = (fund * finalNukeFactor) / MAX_DENOMINATOR; // Calculate the potential claim amount based on the finalNukeFactor
    uint256 maxAllowedClaimAmount = fund / maxAllowedClaimDivisor; // Define a maximum allowed claim amount as 50% of the current fund size

    // Directly assign the value to claimAmount based on the condition, removing the redeclaration
    uint256 claimAmount = finalNukeFactor > nukeFactorMaxParam
      ? maxAllowedClaimAmount
      : potentialClaimAmount;
+   require(claimAmount >= minimumClaim, "Claimable amount is below the minimum);
    fund -= claimAmount; // Deduct the claim amount from the fund

    nftContract.burn(tokenId); // Burn the token
    (bool success, ) = payable(msg.sender).call{ value: claimAmount }('');
    require(success, 'Failed to send Ether');

    emit Nuked(msg.sender, tokenId, claimAmount); // Emit the event with the actual claim amount
    emit FundBalanceUpdated(fund); // Update the fund balance
  }
```

# M-05 Incorrect price calculation for first token of each generation due to delayed generation increment
## Impact
After incrementing each generation, the `priceIncrement` is incremented by `priceIncrementByGen` in [_incrementGeneration](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/TraitForgeNft/TraitForgeNft.sol#L352). The generation incrementation occurs when minting the first token of the new generation, not when minting the last token of the current generation. This leads to the issue that when minting the first token of each new generation, the `priceIncrementByGen` is not included in the price because `calculateMintPrice` is called before the generation is incremented:
```solidity
function mintToken(
    bytes32[] calldata proof
  )
    public
    payable
    whenNotPaused
    nonReentrant
    onlyWhitelisted(proof, keccak256(abi.encodePacked(msg.sender)))
  {
@>    uint256 mintPrice = calculateMintPrice(); // @audit getting the mint price
    require(msg.value >= mintPrice, 'Insufficient ETH send for minting.');

@>    _mintInternal(msg.sender, mintPrice); // @audit minting token and incrementing generation if necessary

    uint256 excessPayment = msg.value - mintPrice;
    if (excessPayment > 0) {
      (bool refundSuccess, ) = msg.sender.call{ value: excessPayment }('');
      require(refundSuccess, 'Refund of excess payment failed.');
    }
  }

```
When minting the first token of a new generation, the current generation is incremented **after** calculating the minting price because the increment occurs inside `_mintInternal`:
```solidity
function _mintInternal(address to, uint256 mintPrice) internal {
    if (generationMintCounts[currentGeneration] >= maxTokensPerGen) {
      _incrementGeneration();
    }
    //...
}
```
As a result, the required mint price for the first token of each new generation does not include the `priceIncrementByGen`.

## Tools Used
Manual Review

## Recommended Mitigation Steps
The root cause of this issue is that the generation incrementation occurs when minting the first token of the new generation, rather than when minting the last token of the current generation. A simple fix is to move the generation incrementation check to the end of the mint process. Below is a suggestion for updating the code in `mintToken` and `_mintInternal`
```diff
// mintToken

function mintToken(
    bytes32[] calldata proof
  )
    public
    payable
    whenNotPaused
    nonReentrant
    onlyWhitelisted(proof, keccak256(abi.encodePacked(msg.sender)))
  {
    uint256 mintPrice = calculateMintPrice();
    require(msg.value >= mintPrice, 'Insufficient ETH send for minting.');

    _mintInternal(msg.sender, mintPrice);

    uint256 excessPayment = msg.value - mintPrice;
    if (excessPayment > 0) {
      (bool refundSuccess, ) = msg.sender.call{ value: excessPayment }('');
      require(refundSuccess, 'Refund of excess payment failed.');
    }

+    if (generationMintCounts[currentGeneration] == maxTokensPerGen) {
+       _incrementGeneration();
+    }
  }

// _mintInternal
function _mintInternal(address to, uint256 mintPrice) internal {
-    if (generationMintCounts[currentGeneration] >= maxTokensPerGen) {
-      _incrementGeneration();
-   }
    //...
}
```