# Blend Whitepaper

## Abstract

This paper introduces the first decentralized money market primitive, Blend. A money market primitive enables the permissionless creation of lending markets to quickly respond to emerging market needs. Applications, industries, and users can utilize this primitive to create isolated lending pools that best serve their niche. Blend is an ungoverned, modular, money market primitive that does not compromise on security or capital efficiency.

## Introduction

Decentralized money markets act as a cornerstone for healthy crypto-economic systems. They trustlessly facilitate the flow of capital to wherever it is most efficient, increasing capital efficiency and generating interest along the way. Aave and Compound prove how valuable these products are with their success in the Ethereum ecosystem. Since their inception during the early stages of decentralized finance (DeFi), they quickly became two of the industry’s largest and most used DeFi protocols. Aave remains one of the largest, peaking at approximately $30 billion in liquidity [1].

Despite their usefulness, current money markets fall short in terms of flexibility. Users want to utilize a wide range of their crypto assets in money markets. However, supporting risky assets, especially as collateral, can put protocol funds at risk. Aave and Compound forgo flexibility and have extensive governance systems that ensure any asset meets well-defined criteria before adding it to their markets [2, 3]. Other protocols like Euler and Rari have novel approaches for managing permissionless listings that segment asset risk [4]. Unfortunately, these approaches can lead to liquidity fragmentation and low capital utilization.

Blend represents a new, more primitive approach to decentralized money market protocols. It enables the permissionless creation of isolated lending pools, ensuring maximum flexibility and accessibility. Blend avoids the normal pitfalls of permissionless lending markets by using a market-driven backstop system to curate pools, ensuring appropriate risk levels, and using a dynamic interest rate model to preserve capital efficiency.

## Protocol Specification

### System Diagram

`pretty pic here`\

### Isolated Lending Pools

Blend’s core component is its isolated lending pools. These pools facilitate lending and borrowing between users for a set of supported assets. Anyone can deploy an isolated pool using Blend, allowing the protocol to quickly adapt to the needs of constantly evolving digital economies. The deployer sets parameters that govern the pool’s supported assets, acceptable loan-to-value ratios, target utilization rates, and price oracle.

Blend ensures asset risk segmentation by logically separating positions in isolated pools from all other pools, meaning collateral and debt associated with one isolated pool does not apply to others. Thus, bad debt, liquidations, or bad oracles in one pool cannot spill over and harm users in another pool. As a result, the adaptability offered by Blend’s permissionless deployment does not expose protocol users to unknown risks.

There are two types of isolated pools, standard and owned:

#### Standard Pools

Standard pool parameters are immutable after deployment. As such, any parameters defined by the deployer, like the support assets, are fixed. This creates a trust-minimized money market environment, lowering the risk-surface for users.

#### Owned Pools

Owned pools are isolated lending pools where a delegated address can modify pool state and most pool parameters. Notably, they cannot modify the oracle contract address parameter or the backstop take rate parameter. This restriction prevents excessive damage to users by malicious or compromised pool owners. Owned pools are attractive choices for DAOs or platforms that want to offer their own lending market.

### BLND Token

BLND is Blend’s protocol token; its primary purpose is backstopping and managing lending markets through the backstop module.\

### Backstop Module

The backstop module is a pool of funds that acts as first-loss capital for each isolated lending pool. Each pool's backstop funds are specific to themselves, and used extensively in pool management.

#### Depositing and Withdrawing Funds

Users can deposit BLND or BLND:USDC liquidity pool tokens into a backstop module for an isolated lending pool. They can withdraw their deposits at any time. However, initiating a withdrawal places the funds into a withdrawal queue, where they remain for 30 days. After the queue expires, users can withdraw their funds as long as the backstop module has no remaining bad debt. The queue period ensures the backstop module can effectively perform its function as lender insurance.

#### Lending Pool Interest Sharing

In exchange for insuring pools, backstop module depositors receive a portion of the interest paid by pool borrowers unless their deposit is currently queued for withdrawal. The percent of borrower interest paid to the backstop module depends on the BackstopTakeRate parameter, which is set on pool creation and validated such that it is set within [0.05, 0.7]. The portion of interest paid to backstop module depositors should be set higher for high-risk pools and lower for low-risk pools, reflecting their differing insurance requirements.

#### Covering Bad Debt

Pool backstop modules act as first-loss capital by assuming any bad debt the pool takes on, using the pool’s backstop deposits as collateral. Backstop module deposits have a global collateral factor of 0.8. If a backstop module assumes too much bad debt for its deposits to collateralize, the backstop module can be liquidated. Any remaining bad debt is socialized among lenders if backstop module deposits are insufficient to complete the liquidation.

#### Repaying Bad Debt

When a backstop module has taken on bad debt, it attempts to repay it using the interest that would normally be distributed to backstop module depositors. The backstop module will accrue interest to a bad debt repayment fund, and at any point any entity may choose to repay all bad debt accrued by the backstop module in exchange for the accrued interest.

#### Self Liquidation

There is a possible scenario where a pool is effectively abandoned but still has bad debt. This circumstance obstructs all backstop module withdrawals as withdrawals cannot be completed while the backstop is indebted. However, since the pool has been abandoned (all lenders and borrowers have withdrawn funds), it will be impossible for backstop module depositors to generate enough interest to repay the bad debt. To prevent this stalemate, the backstop module includes a function that initiates a liquidation auction for the backstop module if 70% of backstop deposits have completed their withdrawal queue and are waiting to be withdrawn.

### Lending and Borrowing

#### Lending Assets

Any asset supported by a given pool can be deposited into that pool. A deposit is represented by an ERC-20 token, or bToken, which entitles the holder to a share of the total deposited assets (similar to Compound’s cTokens [5]).

The lender receives bTokens for depositing amount of an asset based on the following exchange rate:\

    bTokenLenderRate = 1+(bTokenRate-1)*(1-BackstopTakeRate)

    bTokens = amount/bTokenLenderRate

The bTokenRate is an internal value that tracks how much interest has accrued to one bToken over the pool’s lifetime. For example, if 10% interest has been generated by a bToken, its bTokenRate will be 1.1. bTokenLenderRate is the amount of interest that has been allocated to lenders from the bTokenRate.

Lenders can include a “sponsor” field on their deposit transaction to turn it into a sponsored deposit. Including a sponsor causes the transaction to send up to 0.5% (based on pool settings) of the resulting bTokens to the specified sponsor. This functionality incentivizes wallets and other entities to host Blend UIs, as they can charge a small deposit fee.

#### Borrowing Assets

Any asset supported by the pool can be borrowed from the pool as long as the borrower has sufficient collateral deposited. In the event of a price change, borrowers risk liquidation if the collateral they have posted is no longer sufficient to cover their outstanding liabilities.

Borrowed assets are tracked with an ERC-20 token, or dToken (similar to Aave’s debtToken), which represents an outstanding liability for the holder against the pool for the borrowed token. These tokens are non-transferable and can only be removed by repaying the borrowed amount to the pool.

The borrower receives dTokens for borrowing amount of an asset from the pool based on the following exchange rate:

$dTokenRate = \frac{(bTokenTotalSupply * bTokenRate - PoolAssetBalance)}{dTokenTotalSupply}$

$dTokens=\frac{amount}/dTokenRate

The value of assets a user can borrow from a pool is based on their Borrow Limit. Each collateral position in the pool increases their borrow limit by the value of the position multiplied by its Collateral Factor. Each liability position decreases their borrow limit by the value of the position divided by its Liability Factor. Each asset's collateral and liability factor is set on pool creation and bounded within [0.00, 1.00].

$Borrow Limit = \sum(CollateralPositionValue * AssetCollateralFactor) -\sum(\frac{LiabilityPositionValue}{LiabilityFactor})$

Supporting both a collateral and liability factor gives pool creators a large amount of flexibility regarding the level of leverage they permit for different positions. For example, if a pool creator wants to support high leverage for fiat borrows collateralized with fiat, but not crypto borrows collateralized with fiat, they can set all fiat collateral and liability factors to 0.98 and crypto collateral and liability factors to 0.765. This would give fiat borrowers using fiat as collateral up to 25x leverage, but crypto borrowers using fiat as collateral only up to 4x leverage. This level of flexibility is not possible using only collateral factors.

### Interest Rates

Each pool algorithmically sets each of its assets’ interest rates based on each asset’s utilization ratio. The interest rate adjusts dynamically to stabilize the utilization to a constant target utilization ratio. The utilization ratio of an asset is defined by:

U=1- BalancePoolbTokenTotalSupply \* bTokenRate

Each asset in the pool defines a target utilization rate and three initial interest rates: at the target utilization ratio (UT), 95% utilization ratio, and 100% utilization ratio. The initial rates are used to calculate three slope values R1, R2, and R3. These values define an interest rate model, similar to Aave’s[6], but with three distinct legs:

    IR = RM*(Rbase+UUTR1), where U UT

IR = RM\*(Rbase+R1+U-UT0.95-UTR2), where UT<U0.95
IR = RM\*(Rbase+R1+R2)+U-0.950.05R3, where 0.95<U

    Where:

IR = Interest rate
RM = Rate modifier
U = Utilization ratio
Rbase = Base protocol interest rate (0.01)

Figure 1: Interest rate curve examples for various asset classes where low curve has (UT=0.9, R1=0.03, R2=0.2, R3=1), medium curve has (UT=0.75, R1=0.05, R2=0.5, R3=1.5), high curve has (UT=0.6, R1=0.07, R2=1, R3=2), and the Rate Modifier is 1.

The Rate Modifier is a reactive value that adjusts the interest rate the pool charges for borrowing the asset to achieve a target utilization rate. The modifier slowly sums the error in utilization over blocks, compensating for any steady-state error as a result of the user-defined initial interest rates. The value is bounded between [0.1, 100] to limit the effective interest rate change possible and avoid potential integral windup that could cause instability. The updated modifier is calculated as follows:

Util Rate Error = blocks * (UT-U)
Rate Modifier=Util Rate Error *Reactivity Constant+Rate Modifier

The Target Utilization Ratio and Reactivity Constant are constant values provided on pool creation for each asset supported by the pool. They are validated such that Target Utilization Rate is within [0.5, 0.95] and Reactivity Constant is within [10e-6, 10e-4].

Interest rates are accrued discretely over the blocks since the last accrual has occurred. The following formula depicts how a loan value will get updated for a given interest rate, R:

Loan Value = Loan Value _ (1+R_ blocksavg blocks per year)

Accruing discretely significantly lessens the gas required to accrue a given liability, and only slightly underestimates the full liability compared to a continuously compounding loan over a ten year time frame. Thus, it is an appropriate trade off for keeping contract calls cheap.

This dynamic interest rate model responds more efficiently to changing cost of capital in the crypto lending space than a traditional interest curve model. For example, if USDC borrowing rates fall in other protocols, we can also expect the utilization of USDC in Blend markets to fall. Thanks to the dynamic interest rate model, the Blend markets will automatically adjust their Rate Modifier for USDC, lowering interest rates and driving utilization rates back up. Thus, Blend’s interest rate model enables markets to retain their goal utilization ratios and levels of capital efficiency without requiring any intervention from a governance system.

### Liquidations

In the event a borrower’s outstanding liabilities to a given pool exceed the borrowing capacity of their collateral in that pool, the borrower can be liquidated. Liquidations are performed to reduce the chance a borrower defaults on their loan, causing a permanent loss of lender funds.

Blend uses gentle dutch auctions to liquidate borrowers without overpenalizing them or exposing them to oracle risk. The auctions are carried out in three steps:

**Auction Initiation**\
Any user can initiate a liquidation auction on an account which has exceeded its borrow capacity for a pool. The specific positions involved in the auction are the user’s largest collateral position in the pool and their largest liability position in the pool. Limiting auctions to two positions simplifies auctions for liquidators and removes potential auction manipulation vectors. The downside is multiple auctions may be required to fully liquidate some users.

**Auction Duration**\
After the auction initiation, an Ask Modifier value begins scaling from zero to one based on how much time has passed since initiation. This value governs the percent of the user’s collateral position the liquidator will receive upon filling the auction (1.00 being 100% of the position). Once the Ask Modifier reaches one, a Bid Modifier value begins decreasing from one to zero. The Bid Modifier governs what percentage of the Target Liquidation Amount the liquidator must provide when they fill the auction (1.00 being 100%).

Ask Modifier Formula:
⌊ bc-bi2⌋100bc 200
1bc> 200
Bid Modifier Formula:
1bc 200
⌊ bi - bc2⌋100+2bC> 200
Where:
bi = Initiation block
bc = Current block
Auction Fill
At any point, a liquidator can fill the auction and repay a portion of the user’s liability in exchange for a portion of their collateral. The amount of the user’s liability which must be repaid is the Target Liquidation Amount. It is calculated when the auction is filled using the formula below. If the calculated amount is larger than the liability position associated with the auction, the full liability position is treated as the target liquidation amount.

Target Liquidation Amount Formula:

B*1.02*LO-C\*K1.02O-2K1+OK
Where:
B = Bid modifier
L = Total value of the user’s liability positions in the lending pool
O = Liability Factor of the liability being repaid by the liquidator
O = Average Liability Factor of the user’s liability positions in the lending pool
C = Total value of the user’s collateral positions in the lending pool
K = Collateral Factor of the collateral being auctioned
K = Average Collateral Factor of all the user’s collateral positions in the lending pool
Price Oracles
Price oracles are used extensively for determining if a borrower’s outstanding and potential liabilities are sufficiently collateralized or not. Each isolated lending pool specifies a price oracle to use on creation. The pool creator can select any deployed oracle contract as long as the oracle implements the expected BlendOracle trait, and all assets supported by the pool can fetch USD-denominated prices from the oracle.
Governance and Decentralization
Overview
Blend is an ungoverned protocol. That is, no DAO or organization is required to manage the protocol and BLND does not have voting power. Instead, social governance and market forces surrounding backstop modules drive changes.

DAOs are historically bureaucratic, making it difficult for them to support the speed and flexibility required by the DeFi ecosystem. The primary reason DAOs are used by lending protocols is their ability to control and underwrite risk. Blend’s ungoverned model achieves similar levels of risk management using distributed market consensus rather than explicit DAO consensus. Thus, Blend can eschew the traditional DAO model in favor of a market-driven approach that allows it to respond much faster to market changes.
Pool Management
Besides insuring pools, backstop modules are crucial for curating and managing pools. They perform this function by triggering modification of their pool’s state, which governs whether the pool is active (whether borrowing and depositing is allowed) or not.
Pool State
Blend’s isolated lending pools have three possible states that determine how they function:

Active: All operations are enabled
On Ice: Only borrowing is disabled
Frozen: Both borrowing and depositing are disabled

Pools start on ice, and their state can change depending on the state of their respective backstop module.

Active can be set if less than 25% of backstop module deposits are in the withdrawal queue and module deposits net queued withdrawals are above the backstop threshold.
On Ice can be set if 25% or more of the backstop module deposits are in the withdrawal queue or if module deposits, including queued withdrawals, are under the backstop threshold.
Frozen can be set if 50% or more of the backstop module deposits are in the withdrawal queue.

Pool state is modified based on Backstop module status because backstop module depositors are first-loss capital for pools and thus are the most exposed to pool risk levels. If pool risk levels change, it is expected backstop module depositors will be the first to react.

In the case of owned pools, the pool owner can set their pool to on ice or frozen, as they are also expected to monitor pool risk levels.
Backstop Threshold
The backstop deposit threshold required to activate pools is 1.5 million BLND tokens. Each BLND token in BLND:USDC liquidity pool token deposits count as two tokens toward this threshold. BLND-only deposits count as 0.5 tokens. The reason for this varied weight is because pure BLND tokens are not as effective at insuring the pool as BLND:USDC liquidity pool tokens are.
Pool Migration
Due to the immutable nature of standard isolated lending pools, if new parameters are required, a new pool must be deployed, and users must be migrated to it. Migrations are especially urgent if the old pool is using an oracle contract that is being shut down. To migrate users, someone first needs to create the new pool, then inform the backstop module depositors of the old pool what the new pool is and why they should migrate. Since an outdated parameter could jeopardize backstop module deposits, depositors should be more than happy to migrate. Once they begin the migration process by queueing their deposits for withdrawal, the pool can be turned off using pool management functions. This will force lenders and borrowers to migrate to the new pool to continue their operations.
BLND Emissions
BLND tokens are emitted by the protocol to users. An emissions contract controls all protocol emissions, and the backstop module determines how they get distributed. In total, the protocol emits 500,000 BLND tokens weekly. The tokens are distributed to two types of users: backstop depositors and isolated pool borrowers.
Backstop Depositor Emissions
Users who deposit BLND:USDC LP tokens into the backstop module are eligible to receive a share of 70% of the total BLND emissions. All BLND:USDC LP token deposits are eligible for emissions regardless of the isolated pool the token is deposited in or the withdrawal queue status. Each BLND:USDC LP token deposit receives emissions proportional to the total BLND:USDC LP tokens deposited by all users in the backstop module.
Isolated Pool Borrower Emissions
Users who borrow from active Isolated Pools are eligible to receive a share of 30% of the total BLND emissions. The backstop module splits the share of weekly emissions among these pools proportionally to their backstop sizes. From there, pools split their emission allowance to all borrowers based on the assets emission rate. When these emissions are claimed, they are deposited into the backstop module of the pool the user was borrowing from.
Emissions as Balancing Force
The protocol sending emissions to pool borrowers creates a necessary correlation between the pool’s backstop size and the pool’s total value locked (TVL). If the pool’s backstop module size outpaces the pool’s TVL, more emissions will be directed towards the borrowers of the pool. This incentivizes more borrowing, and, by effect, more lending, which increases the pool’s TVL. If the pool’s TVL outpaces the pool’s backstop module, more interest is generated for backstop module depositors. This incentivizes more deposits into the pool’s backstop module, increasing its size. As a result of these incentive mechanisms, a pool’s backstop module size and its TVL remain correlated regardless of fluctuations, protecting lender funds.
Emission Migration
The emission contract is not upgradeable, and is only responsible for minted 500,000 BLND tokens a week. Thus, all protocol logic regarding emissions distribution is the responsibility of the backstop module. In the event a new version of the protocol is released, the emission contract comes with an upgrade function that will redirect emissions from the old backstop module to the new contract. The upgrade function will only change the emission destination if, according to the backstop threshold accounting method, the new contract has a larger backstop size than the current backstop module. Thus, the new contract needs to convince the current backstop module depositors to migrate.

In the event a new backstop module contract is set, all currently existing pools will no longer receive emissions until they perform a backstop module update. After this is completed, the new backstop module assumes the responsibility of first-loss capital and the pool will function as expected.
References
https://github.com/aave/aave-v3-core/blob/master/techpaper/Aave_V3_Technical_Paper.pdf
https://medium.com/gauntlet-networks/gauntlets-parameter-recommendation-methodology-8591478a0c1c
https://docs.aave.com/risk/asset-risk/introduction
https://docs.euler.finance/getting-started/white-paper#permissionless-listing
https://compound.finance/docs/ctokens
https://github.com/aave/aave-protocol/blob/master/docs/Aave_Protocol_Whitepaper_v1_0.pdf
