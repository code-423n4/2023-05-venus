# Venus Protocol contest details
- Total Prize Pool: $90,500 USDC
  - HM awards: $56,250 USDC 
  - QA report awards: $7,500 USDC 
  - Gas report awards: $3,750 USDC
  - Bot Race awards: $7,500 USDC 
  - Judge awards: $9,000 USDC
  - Lookout awards: $6,000 USDC 
  - Scout awards: $500 USDC 
- Join [C4 Discord](https://discord.gg/code4rena) to register
- Submit findings [using the C4 form](https://code4rena.com/contests/2023-05-venus-contest/submit)
- [Read our guidelines for more details](https://docs.code4rena.com/roles/wardens)
- Starts May 05, 2023 20:00 UTC
- Ends May 12, 20:00 UTC

## Automated Findings / Publicly Known Issues

Automated findings output for the contest can be found [here](add link to report) within 24 hours of contest opening.

*Note for C4 wardens: Anything included in the automated findings output is considered a publicly known issue and is ineligible for awards.*

# Overview

[Venus](https://app.venus.io) is a decentralized finance (DeFi) algorithmic money market protocol on BNB Chain.

Decentralized lending pools are very similar to traditional lending services offered by banks, except that they are offered by P2P decentralized platforms. Users can leverage assets by borrowing and lending assets listed in a pool. Lending pools help crypto holders earn a substantial income through interest paid on their supplied assets and access assets they don't currently own without selling any of their portfolio.

The first generation of lending pools, including the Venus Core Pool, aggregate a diverse group of assets for users to interact with. This introduces significant risk to the entire protocol's liquidity if any token included in the pool experiences extreme volatility. It is difficult to list new tokens due to the lack of specific risk parameters. The Venus Core Pool also lacks risk analysis parameters for the pool's shortfall which would give users a view into the current market stability.

**Venus Isolated Pools** is designed to tackle all of the shortcomings that its previous versions had. Isolated Pools is composed of independent collections of assets with custom risk management configurations, giving users broader opportunities to manage risk, allocate their assets in the protocol and earn yield. Multiple isolated pools also reduce the impact of any potential asset failure affecting the liquidity of the protocol. Rewards can be customized per market in each pool to provide the best incentives to users.

## Contract Summaries

### PoolRegistry

The Isolated Pools architecture centers around the `PoolRegistry` contract. The `PoolRegistry` maintains a directory of isolated lending pools and can perform actions like creating and registering new pools, adding new markets to existing pools, setting and updating the pool's required metadata, and providing the getter methods to get information on the pools.

![image](https://user-images.githubusercontent.com/47150934/236290058-6b14a499-7afe-46e4-bca6-d72e3db8a28e.png)

### Factories

There are three factory contracts:

* JumpRateModelFactory
* VTokenProxyFactory
* WhiteRateInterestModelFactory

These contracts are designed to deploy contracts when adding markets to a pool. In particular, the `JumpRateModelFactory` and `WhiteRateInterestModelFactory` deploy contracts for the rate model based on which was chosen for the market. The `VtokenProxyFactory` is used to generate a new `vToken` proxy for each market when it is added to a pool.

### Risk Fund

The risk fund concerns three main contracts:

* ProtocolShareReserve
* RiskFund
* ReserveHelpers

The three contracts are designed to hold fees that have been accumulated from liquidations and spread, send a portion to the protocol treasury, and send the remainder to the RiskFund. When `reduceReserves()` is called in a `vToken` contract, all accumulated liquidation fees and spread are sent to the `ProtocolShareReserve` contract. Once funds are transferred to the `ProtocolShareReserve`, anyone can call `releaseFunds()` to transfer 70% to the `protocolIncome` address and the other 30% to the `riskFund` contract. Once in the `riskFund` contract, the tokens can be swapped via `PancakeSwap` pairs to the convertible base asset, which can be updated by the owner of the contract. When tokens are converted to the `convertibleBaseAsset`, they can be used in the `Shortfall` contract to auction off the pool's bad debt. Note that just as each pool is isolated, the risk funds for each pool are also isolated: only the risk fund for the same pool can be used when auctioning off the bad debt of the pool.

### Shortfall

`Shortfall` is an auction contract designed to auction off the `convertibleBaseAsset` accumulated in `RiskFund`. The `convertibleBaseAsset` is auctioned in exchange for users paying off the pool's bad debt. An auction can be started by anyone once a pool's bad debt has reached a minimum value. This value is set and can be changed by the authorized accounts. If the pool’s bad debt exceeds the risk fund plus a 10% incentive, then the auction winner is determined by who will pay off the largest percentage of the pool's bad debt. The auction winner then exchanges for the entire risk fund. Otherwise, if the risk fund covers the pool's bad debt plus the 10% incentive, then the auction winner is determined by who will take the smallest percentage of the risk fund in exchange for paying off all the pool's bad debt.

### Rewards

Users can receive additional rewards through a `RewardsDistributor`. Each `RewardsDistributor` proxy is initialized with a specific reward token and `Comptroller`, which can then distribute the reward token to users that supply or borrow in the associated pool. Authorized users can set the reward token borrow and supply speeds for each market in the pool. This sets a fixed amount of reward token to be released each block for borrowers and suppliers, which is distributed based on a user’s percentage of the borrows or supplies respectively. The owner can also set up reward distributions to contributor addresses (distinct from suppliers and borrowers) by setting their contributor reward token speed, which similarly allocates a fixed amount of reward token per block.

The owner has the ability to transfer any amount of reward tokens held by the contract to any other address. Rewards are not distributed automatically and must be claimed by a user calling `claimRewardToken()`. Users should be aware that it is up to the owner and other centralized entities to ensure that the `RewardsDistributor` holds enough tokens to distribute the accumulated rewards of users and contributors.

### PoolLens

The `PoolLens` contract is designed to retrieve important information for each registered pool. A list of essential information for all pools within the lending protocol can be acquired through the function `getAllPools()`. Additionally, the following records can be looked up for specific pools and markets:

* the vToken balance of a given user;
* the pool data (oracle address, associated vToken, liquidation incentive, etc) of a pool via its associated comptroller address;
* the vToken address in a pool for a given asset;
* a list of all pools that support an asset;
* the underlying asset price of a vToken;
* the metadata (exchange/borrow/supply rate, total supply, collateral factor, etc) of any vToken.

### Rate Models

These contracts help algorithmically determine the interest rate based on supply and demand. If the demand is low, then the interest rates should be lower. In times of high utilization, the interest rates should go up. As such, the lending market borrowers will earn interest equal to the borrowing rate multiplied by utilization ratio.

### VToken

Each asset that is supported by a pool is integrated through an instance of the `VToken` contract. As outlined in the protocol overview, each isolated pool creates its own `vToken` corresponding to an asset. Within a given pool, each included `vToken` is referred to as a market of the pool. The main actions a user regularly interacts with in a market are:

* mint/redeem of vTokens;
* transfer of vTokens;
* borrow/repay a loan on an underlying asset;
* liquidate a borrow or liquidate/heal an account.

A user supplies the underlying asset to a pool by minting `vTokens`, where the corresponding `vToken` amount is determined by the `exchangeRate`. The `exchangeRate` will change over time, dependent on a number of factors, some of which accrue interest. Additionally, once users have minted `vToken` in a pool, they can borrow any asset in the isolated pool by using their `vToken` as collateral. In order to borrow an asset or use a `vToken` as collateral, the user must be entered into each corresponding market (else, the `vToken` will not be considered collateral for a borrow). Note that a user may borrow up to a portion of their collateral determined by the market’s collateral factor. However, if their borrowed amount exceeds an amount calculated using the market’s corresponding liquidation threshold, the borrow is eligible for liquidation. When a user repays a borrow, they must also pay off interest accrued on the borrow.

The Venus protocol includes unique mechanisms for healing an account and liquidating an account. These actions are performed in the `Comptroller` and consider all borrows and collateral for which a given account is entered within a market. These functions may only be called on an account with a total collateral amount that is no larger than a universal `minLiquidatableCollateral` value, which is used for all markets within a `Comptroller`. Both functions settle all of an account’s borrows, but `healAccount()` may add `badDebt` to a vToken. For more detail, see the description of `healAccount()` and `liquidateAccount()` in the `Comptroller` summary section below.

### Comptroller

The `Comptroller` is designed to provide checks for all minting, redeeming, transferring, borrowing, lending, repaying, liquidating, and seizing done by the `vToken` contract. Each pool has one `Comptroller` checking these interactions across markets. When a user interacts with a given market by one of these main actions, a call is made to a corresponding hook in the associated `Comptroller`, which either allows or reverts the transaction. These hooks also update supply and borrow rewards as they are called. The comptroller holds the logic for assessing liquidity snapshots of an account via the collateral factor and liquidation threshold. This check determines the collateral needed for a borrow, as well as how much of a borrow may be liquidated. A user may borrow a portion of their collateral with the maximum amount determined by the markets collateral factor. However, if their borrowed amount exceeds an amount calculated using the market’s corresponding liquidation threshold, the borrow is eligible for liquidation.

The `Comptroller` also includes two functions `liquidateAccount()` and `healAccount()`, which are meant to handle accounts that do not exceed the `minLiquidatableCollateral` for the `Comptroller`:

* `healAccount()`: This function is called to seize all of a given user’s collateral, requiring the `msg.sender` repay a certain percentage of the debt calculated by `collateral/(borrows*liquidationIncentive)`. The function can only be called if the calculated percentage does not exceed 100%, because otherwise no `badDebt` would be created and `liquidateAccount()` should be used instead. The difference in the actual amount of debt and debt paid off is recorded as `badDebt` for each market, which can then be auctioned off for the risk reserves of the associated pool.

* `liquidateAccount()`: This function can only be called if the collateral seized will cover all borrows of an account, as well as the liquidation incentive. Otherwise, the pool will incur bad debt, in which case the function `healAccount()` should be used instead. This function skips the logic verifying that the repay amount does not exceed the close factor.

# Scope

| File | SLOC | Purpose | Libraries used |  
| ----------- | ----------- | ----------- | ----------- |
| [Comptroller.sol](https://github.com/code-423n4/2023-05-venus/blob/main/contracts/Comptroller.sol) | 728 | Provide checks for all minting, redeeming, transferring, borrowing, lending, repaying, liquidating, and seizing done by the `vToken` contract | [@openzeppelin/contracts-upgradeable/access/Ownable2StepUpgradeable.sol](https://www.openzeppelin.com/contracts) [@venusprotocol/oracle/contracts/PriceOracle.sol](https://github.com/VenusProtocol/oracle) [@venusprotocol/governance-contracts/contracts/Governance/AccessControlManager.sol](https://github.com/VenusProtocol/governance-contracts/blob/main/contracts/Governance/AccessControlManager.sol) |
| [VToken.sol](https://github.com/code-423n4/2023-05-venus/blob/main/contracts/VToken.sol) | 655 | Each VToken instance is a market in the protocol, they hold the underlying assets | [@openzeppelin/contracts-upgradeable/access/Ownable2StepUpgradeable.sol](https://www.openzeppelin.com/contracts) [@openzeppelin/contracts-upgradeable/token/ERC20/utils/SafeERC20Upgradeable.sol](https://www.openzeppelin.com/contracts) [@venusprotocol/governance-contracts/contracts/Governance/AccessControlledV8.sol](https://github.com/VenusProtocol/governance-contracts/blob/main/contracts/Governance/AccessControlledV8.sol) |
| [Lens/PoolLens.sol](https://github.com/code-423n4/2023-05-venus/blob/main/contracts/Lens/PoolLens.sol) | 378 | Retrieve important information for each registered pool | [@openzeppelin/contracts/token/ERC20/extensions/IERC20Metadata.sol](https://www.openzeppelin.com/contracts) [@venusprotocol/oracle/contracts/PriceOracle.sol](https://github.com/VenusProtocol/oracle) |
| [Shortfall/Shortfall.sol](https://github.com/code-423n4/2023-05-venus/blob/main/contracts/Shortfall/Shortfall.sol) | 289 | Auction off funds accumulated in `RiskFund`. These funds are auctioned in exchange for users paying off the pool's bad debt | [@openzeppelin/contracts-upgradeable/access/Ownable2StepUpgradeable.sol](https://www.openzeppelin.com/contracts) [@openzeppelin/contracts-upgradeable/token/ERC20/utils/SafeERC20Upgradeable.sol](https://www.openzeppelin.com/contracts) [@openzeppelin/contracts-upgradeable/security/ReentrancyGuardUpgradeable.sol](https://www.openzeppelin.com/contracts) [@venusprotocol/oracle/contracts/PriceOracle.sol](https://github.com/VenusProtocol/oracle) [@venusprotocol/governance-contracts/contracts/Governance/AccessControlledV8.sol](https://github.com/VenusProtocol/governance-contracts/blob/main/contracts/Governance/AccessControlledV8.sol) |
| [Rewards/RewardsDistributor.sol](https://github.com/code-423n4/2023-05-venus/blob/main/contracts/Rewards/RewardsDistributor.sol) | 266 | Distribute rewards to borrowers and suppliers, based on a user’s percentage of the borrows or supplies respectively | [@openzeppelin/contracts-upgradeable/access/Ownable2StepUpgradeable.sol](https://www.openzeppelin.com/contracts) [@openzeppelin/contracts-upgradeable/token/ERC20/utils/SafeERC20Upgradeable.sol](https://www.openzeppelin.com/contracts) [@venusprotocol/governance-contracts/contracts/Governance/AccessControlledV8.sol](https://github.com/VenusProtocol/governance-contracts/blob/main/contracts/Governance/AccessControlledV8.sol) |
| [Pool/PoolRegistry.sol](https://github.com/code-423n4/2023-05-venus/blob/main/contracts/Pool/PoolRegistry.sol) | 251 | Track of the pools and the markets that have been added to each pool | [@openzeppelin/contracts-upgradeable/access/Ownable2StepUpgradeable.sol](https://www.openzeppelin.com/contracts) [@openzeppelin/contracts/proxy/beacon/BeaconProxy.sol](https://www.openzeppelin.com/contracts) [@openzeppelin/contracts/token/ERC20/extensions/IERC20Metadata.sol](https://www.openzeppelin.com/contracts) [@venusprotocol/governance-contracts/contracts/Governance/AccessControlManager.sol](https://github.com/VenusProtocol/governance-contracts/blob/main/contracts/Governance/AccessControlManager.sol) [@venusprotocol/oracle/contracts/PriceOracle.sol](https://github.com/VenusProtocol/oracle) |
| [RiskFund/RiskFund.sol](https://github.com/code-423n4/2023-05-venus/blob/main/contracts/RiskFund/RiskFund.sol) | 166 | Hold funds that will be auctioned off in the `Shortfall` to cover the bad debt | [@openzeppelin/contracts-upgradeable/access/Ownable2StepUpgradeable.sol](https://www.openzeppelin.com/contracts) [@openzeppelin/contracts-upgradeable/token/ERC20/utils/SafeERC20Upgradeable.sol](https://www.openzeppelin.com/contracts) [@venusprotocol/governance-contracts/contracts/Governance/AccessControlledV8.sol](https://github.com/VenusProtocol/governance-contracts/blob/main/contracts/Governance/AccessControlledV8.sol) |
| [VTokenInterfaces.sol](https://github.com/code-423n4/2023-05-venus/blob/main/contracts/VTokenInterfaces.sol) | 141 | Interface of VToken | [@openzeppelin/contracts-upgradeable/token/ERC20/IERC20Upgradeable.sol](https://www.openzeppelin.com/contracts) |
| [ExponentialNoError.sol](https://github.com/code-423n4/2023-05-venus/blob/main/contracts/ExponentialNoError.sol) | 102 | Functions to work with fixed-precision decimals |  |
| [BaseJumpRateModelV2.sol](https://github.com/code-423n4/2023-05-venus/blob/main/contracts/BaseJumpRateModelV2.sol) | 93 | Abstract contract of interest rate models using a kink | [@venusprotocol/governance-contracts/contracts/Governance/AccessControlledV8.sol](https://github.com/VenusProtocol/governance-contracts/blob/main/contracts/Governance/AccessControlledV8.sol) |
| [ComptrollerInterface.sol](https://github.com/code-423n4/2023-05-venus/blob/main/contracts/ComptrollerInterface.sol) | 62 | Interface of Comptroller contract | [@venusprotocol/oracle/contracts/PriceOracle.sol](https://github.com/VenusProtocol/oracle) |
| [ComptrollerStorage.sol](https://github.com/code-423n4/2023-05-venus/blob/main/contracts/ComptrollerStorage.sol) | 58 | Storage variables of the Comptroller contract | [@venusprotocol/oracle/contracts/PriceOracle.sol](https://github.com/VenusProtocol/oracle) |
| [RiskFund/ProtocolShareReserve.sol](https://github.com/code-423n4/2023-05-venus/blob/main/contracts/RiskFund/ProtocolShareReserve.sol) | 57 | Target of the income generated in the `VToken` contracts | [@openzeppelin/contracts-upgradeable/access/Ownable2StepUpgradeable.sol](https://www.openzeppelin.com/contracts) [@openzeppelin/contracts-upgradeable/token/ERC20/utils/SafeERC20Upgradeable.sol](https://www.openzeppelin.com/contracts) |
| [Factories/VTokenProxyFactory.sol](https://github.com/code-423n4/2023-05-venus/blob/main/contracts/Factories/VTokenProxyFactory.sol) | 43 | Generate a new `vToken` proxy for each market when it is added to a pool | [@openzeppelin/contracts/proxy/beacon/BeaconProxy.sol](https://www.openzeppelin.com/contracts) [@venusprotocol/governance-contracts/contracts/Governance/AccessControlManager.sol](https://github.com/VenusProtocol/governance-contracts/blob/main/contracts/Governance/AccessControlManager.sol) |
| [WhitePaperInterestRateModel.sol](https://github.com/code-423n4/2023-05-venus/blob/main/contracts/WhitePaperInterestRateModel.sol) | 43 | Simple rate model | |
| [ErrorReporter.sol](https://github.com/code-423n4/2023-05-venus/blob/main/contracts/ErrorReporter.sol) | 42 | Definition of errors used by the `VToken` contract | |
| [RiskFund/ReserveHelpers.sol](https://github.com/code-423n4/2023-05-venus/blob/main/contracts/RiskFund/ReserveHelpers.sol) | 37 | Contract extended by `ProtocolShareReserve` and `RiskFund`, adding capabilities to receive funds | [@openzeppelin/contracts-upgradeable/token/ERC20/utils/SafeERC20Upgradeable.sol](https://www.openzeppelin.com/contracts) |
| [JumpRateModelV2.sol](https://github.com/code-423n4/2023-05-venus/blob/main/contracts/JumpRateModelV2.sol) | 21 | Specific rate model using a kink | |
| [Factories/JumpRateModelFactory.sol](https://github.com/code-423n4/2023-05-venus/blob/main/contracts/Factories/JumpRateModelFactory.sol) | 20 | Factory of `JumpRateModelV2` instances | |
| [Pool/PoolRegistryInterface.sol](https://github.com/code-423n4/2023-05-venus/blob/main/contracts/Pool/PoolRegistryInterface.sol) | 20 | Interface of `PoolRegistry` | |
| [MaxLoopsLimitHelper.sol](https://github.com/code-423n4/2023-05-venus/blob/main/contracts/MaxLoopsLimitHelper.sol) | 18 | Abstract contract adding some capabilities to avoid too large loops | |
| [InterestRateModel.sol](https://github.com/code-423n4/2023-05-venus/blob/main/contracts/InterestRateModel.sol) | 15 | Abstract contract used by every rate model | |
| [RiskFund/IRiskFund.sol](https://github.com/code-423n4/2023-05-venus/blob/main/contracts/RiskFund/IRiskFund.sol) | 11 | Interface of `RiskFund` | |
| [IPancakeswapV2Router.sol](https://github.com/code-423n4/2023-05-venus/blob/main/contracts/IPancakeswapV2Router.sol) | 10 | Partial interface of `PancakeswapV2Router` | |
| [Factories/WhitePaperInterestRateModelFactory.sol](https://github.com/code-423n4/2023-05-venus/blob/main/contracts/Factories/WhitePaperInterestRateModelFactory.sol) | 8 | Factory of `WhitePaperInterestRateModel` instances | |
| [Proxy/UpgradeableBeacon.sol](https://github.com/code-423n4/2023-05-venus/blob/main/contracts/Proxy/UpgradeableBeacon.sol) | 7 | Used for testing, it should/will be moved to the `tests` folder | [@openzeppelin/contracts/proxy/beacon/UpgradeableBeacon.sol](https://www.openzeppelin.com/contracts) |
| [RiskFund/IProtocolShareReserve.sol](https://github.com/code-423n4/2023-05-venus/blob/main/contracts/RiskFund/IProtocolShareReserve.sol) | 4 | Interface of `ProtocolShareReserve` | |
| [Shortfall/IShortfall.sol](https://github.com/code-423n4/2023-05-venus/blob/main/contracts/Shortfall/IShortfall.sol) | 4 | Interface of `Shortfall` | |

## Out of scope

The following files/contracts are out of the scope for this audit:

* [contracts/test](https://github.com/code-423n4/2023-05-venus/blob/main/contracts/test)
* External libraries:
  * `@openzeppelin/*`
  * `@venusprotocol/governance-contracts/*`
  * `@venusprotocol/oracle/*`
  * `@venusprotocol/venus-protocol/*`

## Publicly Known Issues

*Note for C4 wardens: publicly known issues are ineligible for awards.*

### Manipulation of the Pancakeswap pairs

Related to `RiskFund.sol#swapPoolsAssets`. We will probably change that code in the future, but the risk of a sandwich attack, for example, could be reduced using a private communication channel with validators.

### Incompatibility with fee on transfer tokens

The protocol is no fully compatible with deflationary/rebase tokens. We maintain a guideline with some rules to check before adding new markets, and these tokens are excluded.

For example, the repayment of the full debt does not work if there is a fee during the token transfer.

### Centralization related risks

Owners and admins will be the Normal timelock contract, that is part of the Governance protocol.

Regarding authorization, we'll use the `AccessControlManager` (ACM) deployed at https://bscscan.com/address/0x4788629abc6cfca10f9f969efdeaa1cf70c23555

In this ACM, only [0x939bd8d64c0a9583a7dcea9933f7b21697ab6396](https://bscscan.com/address/0x939bd8d64c0a9583a7dcea9933f7b21697ab6396) (Normal timelock) has the DEFAULT_ADMIN_ROLE. And this contract is a Timelock contract use during the Venus Improvement Proposals.

There are two other Timelock contracts to execute VIP's with a shorter delay:

* [Normal](https://bscscan.com/address/0x939bd8d64c0a9583a7dcea9933f7b21697ab6396): 24 hours voting + 48 hours delay
* [Fast-track](https://bscscan.com/address/0x555ba73dB1b006F3f2C7dB7126d6e4343aDBce02): 24 hours voting + 6 hours delay
* [Critical](https://bscscan.com/address/0x213c446ec11e45b15a6E29C1C1b402B8897f606d): 6 hours voting + 1 hour delay

### Uncontrolled interactions in `Lens/PoolLens.sol`

Potentially, they could consume more gas than available. We decided not to integrate the [MaxLoopsLimitHelper.sol](https://github.com/code-423n4/2023-05-venus/blob/main/contracts/MaxLoopsLimitHelper.sol) contract in this case to avoid storage variables in the `PoolLens` contract.

### `minLiquidatableCollateral` not limited in `contracts/Comptroller.sol`

So, theoretically a wrong value could be set. But, the value will be set via the Governance, with the user votes, and a timelock contract (see above). So, we don't considered needed to add a extra check.

### `RiskFund._swapAsset`

In this function we are assuming that the `amountOutMin` is defined in USD (`minAmountToConvert` is), but that could be not the case. We should use an oracle here to calculate the value of `amountOutMin` in USD.

Moreover, we are aware of `require(amountOutMin != 0)` is redundant. We'll remove this check.

# Additional Context

*Describe any novel or unique curve logic or mathematical models implemented in the contracts*

*Sponsor, please confirm/edit the information below.*

## Scoping Details 
```
- If you have a public code repo, please share it here:  https://github.com/code-423n4/2023-05-Venus
- How many contracts are in scope?: 23 (`find contracts -name "*.sol" -not -path "contracts/test/*" | xargs grep -e "^abstract contract" -e "^contract" | wc -l`)
- Total SLoC for these contracts?: 3549 (`cloc contracts --exclude-dir=test`)
- How many external imports are there?: 11 (`find contracts -name "*.sol" -not -path "contracts/test/*" | xargs grep -h "import \"@" | sort -u | wc -l `)  
- How many separate interfaces and struct definitions are there for the contracts within scope?: 7 interfaces + 23 structs
- Does most of your code generally use composition or inheritance?: composition
- How many external calls?: 2 (oracles + Pancakeswap)
- What is the overall line coverage percentage provided by your tests?: 94.21
- Is there a need to understand a separate part of the codebase / get context in order to audit this part of the protocol?: no
- Does it use an oracle?: yes (https://github.com/VenusProtocol/oracle)
- Does the token conform to the ERC20 standard?: yes
- Are there any novel or unique curve logic or mathematical models?: no
- Does it use a timelock function?: no
- Is it an NFT?: no
- Does it have an AMM?: no
- Is it a fork of a popular project?: yes, [Compound](https://github.com/compound-finance/compound-protocol)
- Does it use rollups?: no
- Is it multi-chain?: no
- Does it use a side-chain?: no
```

# Tests

## Prerequisites

- NodeJS - 12.x

- Solc - v0.8.13 (https://github.com/ethereum/solidity/releases/tag/v0.8.13)

## Installing

```bash

yarn install

```

## Run Tests

```bash

yarn test

npx hardhat coverage

REPORT_GAS=true npx hardhat test

```

- To run fork tests add FORK_MAINNET=true and QUICK_NODE_KEY in the .env file.

## Deployment

```bash

npx hardhat deploy

```

- This command will execute all the deployment scripts in `./deploy` directory - It will skip only deployment scripts which implement a `skip` condition - Here is example of a skip condition: - Skipping deployment script on `bsctestnet` network `func.skip = async (hre: HardhatRuntimeEnvironment) => hre.network.name !== "bsctestnet";`
- The default network will be `hardhat`
- Deployment to another network: - Make sure the desired network is configured in `hardhat.config.ts` - Add `MNEMONIC` variable in `.env` file - Execute deploy command by adding `--network <network_name>` in the deploy command above - E.g. `npx hardhat deploy --network bsctestnet`
- Execution of single or custom set of scripts is possible, if:
  - In the deployment scripts you have added `tags` for example: - `func.tags = ["MockTokens"];`
  - Once this is done, adding `--tags "<tag_name>,<tag_name>..."` to the deployment command will execute only the scripts containing the tags.

## Source Code Verification

In order to verify the source code of already deployed contracts, run:
`npx hardhat etherscan-verify --network <network_name>`

Make sure you have added `ETHERSCAN_API_KEY` in `.env` file.

## Hardhat Commands

```bash

npx hardhat accounts

npx hardhat compile

npx hardhat clean

npx hardhat test

npx hardhat node

npx hardhat help

REPORT_GAS=true npx hardhat test

npx hardhat coverage

TS_NODE_FILES=true npx ts-node scripts/deploy.ts

npx eslint '**/*.{js,ts}'

npx eslint '**/*.{js,ts}' --fix

npx prettier '**/*.{json,sol,md}' --check

npx prettier '**/*.{json,sol,md}' --write

npx solhint 'contracts/**/*.sol'

npx solhint 'contracts/**/*.sol' --fix



MNEMONIC="<>" BSC_API_KEY="<>" npx hardhat run ./script/hardhat/deploy.ts --network testnet

```

## Documentation

Documentation is autogenerated using [solidity-docgen](https://github.com/OpenZeppelin/solidity-docgen).

They can be generated by running `yarn docgen`

# Links

* Website : https://venus.io
* Twitter : https://twitter.com/venusprotocol
* Telegram : https://t.me/venusprotocol
* Discord : https://discord.com/invite/pTQ9EBHYtF
* Github: https://github.com/VenusProtocol
* Youtube: https://www.youtube.com/@venusprotocolofficial
