# Aave Flash Loan Brownie Mix

![Aave Banner](box-img-sm.png)

## Installation and Setup

1. Clone the Repo 
   
2. [Install Brownie](https://eth-brownie.readthedocs.io/en/stable/install.html) & [Ganache-CLI](https://github.com/trufflesuite/ganache-cli).

3. Sign up for [Infura](https://infura.io/) and generate an API key. Store it in the `WEB3_INFURA_PROJECT_ID` environment variable. You can [learn more about environment variables here](https://www.twilio.com/blog/2017/01/how-to-set-environment-variables.html). Store the API key in the `WEB3_INFURA_PROJECT_ID` environment variable.
   
4. Sign up for [Etherscan](www.etherscan.io) and generate an API key. This is required for fetching source codes of the mainnet contracts we will be interacting with. Store the API key in the `ETHERSCAN_TOKEN` environment variable.

5. Add your `PRIVATE_KEY` environment variable, with [a private key from you wallet.](https://metamask.zendesk.com/hc/en-us/articles/360015289632-How-to-Export-an-Account-Private-Key). *Note: If using metamask, you'll have to add a `0x` to the start of your private key) 


## Testing

To run the tests:

```
source .env
brownie test tests/test_flashloan_v2.py --network mainnet-fork
```
## Basic Console Use

To perform a simple flash loan in a development environment:


1. Run `source .env` in the command
  
2. Open the Brownie console. This automatically launches Ganache on a forked mainnet.

```bash
$ brownie console
```
3. Create variables for the Aave lending pool.

```python
>>> aave_lending_pool_v2 = "0xB53C1a33016B2DC2fF3653530bfF1848a515c8c5"
```

4. Deploy the [`FlashloanV2.sol`](contracts/v2/FlashloanV2.sol) contract.

```python
>>> flashloan = FlashloanV2.deploy(aave_lending_pool_v2, {"from": accounts[0]})
Transaction sent: 0xb0f70b42d2cec9c027b664e9f37490ad50fb934e61f0c58cfe5a77d96dfad681
  Gas price: 0.0 gwei   Gas limit: 12000000   Nonce: 8
  FlashloanV2.constructor confirmed - Block: 11577534   Gas used: 957504 (7.98%)
  FlashloanV2 deployed at: 0x420b1099B9eF5baba6D92029594eF45E19A04A4A
```

5. Transfer some Ether in form of WETH to the newly deployed contract. We must do this because we have not implemented any custom flash loan logic, otherwise the loan will fail from an inability to pay the fee.

```python
>>> WETH = Contract("0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2")
>>> accounts[0].transfer(WETH, "1 ether")
Transaction sent: 0x29ac98c861356bc65e19407d8389e53f8c3d9a05513a8610c9de2ef013aac525
  Gas price: 0.0 gwei   Gas limit: 12000000   Nonce: 10
  Transaction confirmed - Block: 11577536   Gas used: 28431 (0.24%)
>>> WETH.transfer(flashloan, "1 ether", {"from": accounts[0]})
Transaction sent: 0x1bff7e44779eb92d426bc432d22ecb9821e0b66afb66d48404a23e37b34044e6
  Gas price: 0.0 gwei   Gas limit: 12000000   Nonce: 11
  ERC20.transfer confirmed - Block: 11577537   Gas used: 36794 (0.31%)
```

6. Now we are ready to perform our first flash loan!

```python
>>> tx = flashloan.flashloan(WETH, {"from": accounts[0]})
Transaction sent: 0x335530e6d2b7588ee4727b35ae1ed8634a264aca04b325640101ec1c2b89d499
  Gas price: 0.0 gwei   Gas limit: 12000000   Nonce: 12
  FlashloanV2.flashloan confirmed - Block: 11577538   Gas used: 193010 (1.61%)
```

## Implementing Flash Loan Logic

[`contracts/v2/FlashloanV2.sol`](contracts/v2/FlashloanV2.sol) is where you implement your own logic for flash loans. In particular:

* The size of the loan is set in line 89 in `flashloan`.
* Custom flash loan logic is added after line 31 in `executeOperation`.


## Debugging Failed Transactions

Use the `--interactive` flag to open a console immediatly after each failing test:

```
brownie test --interactive
```

Within the console, transaction data is available in the [`history`](https://eth-brownie.readthedocs.io/en/stable/api-network.html#txhistory) container:

```python
>>> history
[<Transaction '0x50f41e2a3c3f44e5d57ae294a8f872f7b97de0cb79b2a4f43cf9f2b6bac61fb4'>,
 <Transaction '0x7af1ce1c30de8b939f481fd6c340226415428f7e6b59e09d7fa5383939091824'>]
```

Examine the [`TransactionReceipt`](https://eth-brownie.readthedocs.io/en/stable/api-network.html#transactionreceipt) for the failed test to determine what went wrong. For example, to view a traceback:

```python
>>> tx = history[-1]
>>> tx.traceback()

Traceback for '0x7af1ce1c30de8b939f481fd6c340226415428f7e6b59e09d7fa5383939091824':


Trace step 13656, program counter 6555:
  File "contracts/protocol/lendingpool/LendingPool.sol", lines 532-536, in LendingPool.flashLoan:    
    IERC20(vars.currentAsset).safeTransferFrom(
      receiverAddress,
      vars.currentATokenAddress,
      vars.currentAmountPlusPremium
    );
Trace step 13750, program counter 11619:
  File "contracts/dependencies/openzeppelin/contracts/SafeERC20.sol", line 36, in SafeERC20.safeTransferFrom:    
    callOptionalReturn(token, abi.encodeWithSelector(token.transferFrom.selector, from, to, value));
Trace step 13937, program counter 16308:
  File "contracts/dependencies/openzeppelin/contracts/SafeERC20.sol", line 55, in SafeERC20.callOptionalReturn:    
    (bool success, bytes memory returndata) = address(token).call(data);
Trace step 13937, program counter 16308:
  File "contracts/dependencies/openzeppelin/contracts/SafeERC20.sol", line 55, in SafeERC20.callOptionalReturn:    
    (bool success, bytes memory returndata) = address(token).call(data);
```

To view a tree map of how the transaction executed:

```python
>>> tx.call_trace()

Call trace for '0x7af1ce1c30de8b939f481fd6c340226415428f7e6b59e09d7fa5383939091824':
Initial call cost  [21432 gas]
FlashloanV2.flashloan  0:14132  [3717 / 174153 gas]
└── ILendingPoolV2.flashLoan  [CALL]  662:14120  [1996 / 170436 gas]
    └── LendingPool.flashLoan  [DELEGATECALL]  759:14107  [218057 / 168440 gas]
        ├── ValidationLogic.validateFlashloan  1428:1461  [106 gas]
        ├── SafeMath.mul  1752:1779  [100 gas]
        ├── SafeMath.div  1783:1831  [161 gas]
        ├── IAtoken  [CALL]  1963:2807  [1914 / 20764 gas]
        │   └── AToken.transferUnderlyingTo  [DELEGATECALL]  2060:2795  [2988 / 18850 gas]
        │       └── ERC20.transfer  [CALL]  2422:2695  [15862 gas]
        ├── FlashloanV2.executeOperation  [CALL]  3252:3904  [3035 / 11055 gas]
        │   └── ERC20.approve  [CALL]  3681:3836  [8020 gas]
        ├── SafeMath.add  4186:4204  [59 gas]
        ├── ReserveLogic.updateState  4270:5480  [10868 / 17605 gas]
        │   ├── InitializableImmutableAdminUpgradeabilityProxy.scaledTotalSupply  [STATICCALL]  4322:4500  [1908 / 3663 gas]
        │   │   └── VariableDebtToken.scaledTotalSupply  [DELEGATECALL]  4419:4488  [1755 gas]
        │   ├── ReserveLogic._updateIndexes  4593:4758  [995 / 1352 gas]
        │   │   ├── MathUtils.calculateLinearInterest  4620:4678  [53 / 183 gas]
        │   │   │   └── SafeMath.sub  4630:4671  [130 gas]
        │   │   ├── WadRayMath.ray  4685:4689  [15 gas]
        │   │   ├── SafeMath.mul  4696:4723  [100 gas]
        │   │   └── SafeMath.add  4732:4750  [59 gas]
        │   ├── WadRayMath.rayMul  4766:4836  [238 gas]
        │   ├── MathUtils.calculateCompoundedInterest  4919:4985  [80 / 210 gas]
        │   │   └── SafeMath.sub  4937:4978  [130 gas]
        │   ├── WadRayMath.rayMul  5018:5088  [238 gas]
        │   ├── WadRayMath.rayMul  5097:5167  [244 gas]
        │   ├── SafeMath.mul  5179:5206  [100 gas]
        │   ├── SafeMath.mul  5210:5237  [100 gas]
        │   ├── SafeMath.mul  5256:5283  [100 gas]
        │   ├── SafeMath.mul  5287:5314  [100 gas]
        │   ├── SafeMath.mul  5318:5345  [100 gas]
        │   ├── SafeMath.mul  5363:5390  [100 gas]
        │   ├── WadRayMath.ray  5394:5398  [15 gas]
        │   ├── SafeMath.add  5402:5420  [59 gas]
        │   ├── SafeMath.add  5424:5442  [59 gas]
        │   └── SafeMath.add  5446:5464  [59 gas]
        ├── WadRayMath.rayMul  5495:5565  [245 gas]
        ├── ReserveLogic._mintToTreasury  5672:7537  [2857 / 14960 gas]
        │   ├── ReserveConfiguration.getReserveFactor  5748:5756  [824 gas]
        │   ├── InitializableImmutableAdminUpgradeabilityProxy.getSupplyData  [STATICCALL]  5820:6706  [1929 / 9318 gas]
        │   │   └── StableDebtToken.getSupplyData  [DELEGATECALL]  5917:6694  [7389 gas]
        │   ├── WadRayMath.rayMul  6813:6883  [245 gas]
        │   ├── WadRayMath.rayMul  6893:6963  [244 gas]
        │   ├── MathUtils.calculateCompoundedInterest  6984:7042  [54 / 191 gas]
        │   │   └── SafeMath.sub  6994:7035  [137 gas]
        │   ├── WadRayMath.rayMul  7075:7145  [244 gas]
        │   ├── WadRayMath.rayMul  7154:7224  [245 gas]
        │   ├── SafeMath.mul  7236:7263  [100 gas]
        │   ├── SafeMath.mul  7267:7294  [100 gas]
        │   ├── SafeMath.mul  7313:7340  [100 gas]
        │   ├── SafeMath.mul  7344:7371  [100 gas]
        │   ├── SafeMath.mul  7375:7402  [100 gas]
        │   ├── SafeMath.mul  7420:7447  [100 gas]
        │   ├── WadRayMath.ray  7451:7455  [15 gas]
        │   ├── SafeMath.add  7459:7477  [59 gas]
        │   ├── SafeMath.add  7481:7499  [59 gas]
        │   └── SafeMath.add  7503:7521  [59 gas]
        ├── WadRayMath.rayMul  7552:7622  [244 gas]
        ├── SafeMath.add  7650:7668  [59 gas]
        ├── SafeMath.sub  7672:7713  [137 gas]
        ├── SafeMath.sub  7724:7765  [137 gas]
        ├── PercentageMath.percentMul  7788:7860  [255 gas]
        ├── IAtoken  [CALL]  7938:8460  [1911 / 19056 gas]
        │   └── AToken.mintToTreasury  [DELEGATECALL]  8035:8448  [17145 gas]
        ├── IAtoken.totalSupply  [STATICCALL]  8530:9167  [1908 / 9796 gas]
        │   └── AToken.totalSupply  [DELEGATECALL]  8627:9155  [3006 / 7888 gas]
        │       └── ILendingPoolV2.getReserveNormalizedIncome  [STATICCALL]  8755:9048  [1911 / 4882 gas]
        │           └── LendingPool.getReserveNormalizedIncome  [DELEGATECALL]  8852:9036  [1237 / 2971 gas]
        │               └── ReserveLogic.getNormalizedIncome  8963:9006  [1734 gas]
        ├── ReserveLogic.cumulateToLiquidityIndex  9246:9616  [2768 / 3626 gas]
        │   ├── WadRayMath.wadToRay  9253:9294  [131 gas]
        │   ├── WadRayMath.wadToRay  9299:9340  [138 gas]
        │   ├── WadRayMath.rayDiv  9344:9423  [270 gas]
        │   ├── WadRayMath.ray  9431:9435  [15 gas]
        │   ├── SafeMath.add  9440:9458  [59 gas]
        │   └── WadRayMath.rayMul  9478:9548  [245 gas]
        ├── ReserveLogic.updateInterestRates  9656:11398  [6759 / 20432 gas]
        │   ├── InitializableImmutableAdminUpgradeabilityProxy.getTotalSupplyAndAvgRate  [STATICCALL]  9767:10627  [1915 / 7639 gas]
        │   │   └── StableDebtToken.getTotalSupplyAndAvgRate  [DELEGATECALL]  9864:10615  [5724 gas]
        │   ├── InitializableImmutableAdminUpgradeabilityProxy.scaledTotalSupply  [STATICCALL]  10769:10947  [1908 / 3663 gas]
        │   │   └── VariableDebtToken.scaledTotalSupply  [DELEGATECALL]  10866:10935  [1755 gas]
        │   ├── WadRayMath.rayMul  11001:11071  [245 gas]
        │   ├── ERC20.balanceOf  [STATICCALL]  11137:11247  [1934 gas]
        │   ├── SafeMath.add  11328:11346  [59 gas]
        │   └── SafeMath.sub  11350:11391  [133 gas]
        ├── ReserveConfiguration.getReserveFactor  11415:11423  [824 gas]
        ├── DefaultReserveInterestRateStrategy.calculateInterestRates  [STATICCALL]  11507:13284  [7330 / 11175 gas]
        │   ├── ILendingPoolAddressesProviderV2.getLendingRateOracle  [STATICCALL]  11838:11954  [1959 gas]
        │   └── LendingRateOracle.getMarketBorrowRate  [STATICCALL]  12031:12129  [1886 gas]
        └── SafeERC20.safeTransferFrom  13657:14107  [291 / -180413 gas]
            └── SafeERC20.callOptionalReturn  13751:14107  [-183382 / -180704 gas]
                ├── Address.isContract  13762:13784  [769 gas]
                └── ERC20.transferFrom  [CALL]  13938:14045  [1909 gas]
```

See the [Brownie documentation](https://eth-brownie.readthedocs.io/en/stable/core-transactions.html) for more detailed information on debugging failed transactions.

# Resources

 - Aave [flash loan documentation](https://docs.aave.com/developers/guides/flash-loans)
 - Aave [Developer Discord channel](https://discord.gg/CJm5Jt3)
 - Brownie [Gitter channel](https://gitter.im/eth-brownie/community)
