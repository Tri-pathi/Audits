# Steadefi - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Depositing to the GMX POOl will return sub-optimal return if the Pool is imbalanced ](#H-01)
    - ### [H-02. Positions may be liquidated due to incorrect implementation of Oracle logic](#H-02)
- ## Medium Risk Findings
    - ### [M-01. Not Choosing optimal path for swaps can result into loss of vaults](#M-01)
    - ### [M-02. Refund Fees will be sent to ](#M-02)
- ## Low Risk Findings
    - ### [L-01. Deposits and Withdrawal from Vault doesn't account deadline for transaction to complete](#L-01)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: Steadefi

### Dates: Oct 26th, 2023 - Nov 6th, 2023

[See more contest details here](https://www.codehawks.com/contests/clo38mm260001la08daw5cbuf)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 2
   - Medium: 2
   - Low: 1


# High Risk Findings

## <a id='H-01'></a>H-01. Depositing to the GMX POOl will return sub-optimal return if the Pool is imbalanced             

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-10-SteadeFi/blob/main/contracts/strategy/gmx/GMXManager.sol#L70

## Summary

Whenever A user deposits tokens to vault, vault create a leverage position depending[delta or delta neutral] in the GMX POOl. performing a proportional deposit is not optimal in every case and depositng tokens in such case will result in fewer LP tokens due to sub optimal trade. Eventually leading to a loss of gain for the strategy vault


## Vulnerability Details

Alice deposits token A() into the vault to make Delta.Neutral position

```solidity
File: GMXVault.sol

  function deposit(GMXTypes.DepositParams memory dp) external payable nonReentrant {
    GMXDeposit.deposit(_store, dp, false);
  }
```
vault refer to deposit to GMXDeposit library to execute the further logic

```solidity
File: GMXDeposit.sol

  function deposit(
    GMXTypes.Store storage self,
    GMXTypes.DepositParams memory dp,
    bool isNative
  ) external {
[................]
    // Borrow assets and create deposit in GMX
    (
      uint256 _borrowTokenAAmt,
      uint256 _borrowTokenBAmt
    ) = GMXManager.calcBorrow(self, _dc.depositValue);

    [.........]
  }
```

https://github.com/Cyfrin/2023-10-SteadeFi/blob/main/contracts/strategy/gmx/GMXDeposit.sol#L54

which calls calcBorrow on GMXManager Library for borrowing assets and making the position IN GMX POOL

```solidity
File: GMXManager.sol

  /**
    * @notice Calculate amount of tokenA and tokenB to borrow
    * @param self GMXTypes.Store
    * @param depositValue USD value in 1e18
  */
  function calcBorrow(
    GMXTypes.Store storage self,
    uint256 depositValue
  ) external view returns (uint256, uint256) {
    // Calculate final position value based on deposit value
    uint256 _positionValue = depositValue * self.leverage / SAFE_MULTIPLIER;

    // Obtain the value to borrow
    uint256 _borrowValue = _positionValue - depositValue;

    uint256 _tokenADecimals = IERC20Metadata(address(self.tokenA)).decimals();
    uint256 _tokenBDecimals = IERC20Metadata(address(self.tokenB)).decimals();
    uint256 _borrowLongTokenAmt;
    uint256 _borrowShortTokenAmt;

 [....................]

    // If delta is neutral, borrow appropriate amount in long token to hedge, and the rest in short token
    if (self.delta == GMXTypes.Delta.Neutral) {
      // Get token weights in LP, e.g. 50% = 5e17
      (uint256 _tokenAWeight,) = GMXReader.tokenWeights(self);

      // Get value of long token (typically tokenA)
      uint256 _longTokenWeightedValue = _tokenAWeight * _positionValue / SAFE_MULTIPLIER;

      // Borrow appropriate amount in long token to hedge
      _borrowLongTokenAmt = _longTokenWeightedValue * SAFE_MULTIPLIER
                            / GMXReader.convertToUsdValue(self, address(self.tokenA), 10**(_tokenADecimals))
                            / (10 ** (18 - _tokenADecimals));

      // Borrow the shortfall value in short token
      _borrowShortTokenAmt = (_borrowValue - _longTokenWeightedValue) * SAFE_MULTIPLIER
                             / GMXReader.convertToUsdValue(self, address(self.tokenB), 10**(_tokenBDecimals))
                             / (10 ** (18 - _tokenBDecimals));
    }
[.....................]
  }
```
https://github.com/Cyfrin/2023-10-SteadeFi/blob/main/contracts/strategy/gmx/GMXManager.sol#L70

Here it consider the current reserve ratio of the pool and deposits in the same ratio.

While GMX docs clearly state that If deposits try to create balance in the pool[depositing in such way which will make actual token weight of index Token towards the TOKEN_WEIGHT defined in the pool] will get benefit technically more LP tokens and oppose to this less LP token if current deposits imbalance the Pool reserve the ratio
[Reference](https://hackmd.io/@0xProtosec/SksD4CY9j#Index)

Even Curve pools work in the same way. Depositer get benefit if they try to balance the pool reserve making them optimal 

## Impact
It is clear that Weight of index token will not be always near equal to the Defined Total_Weight of the Pool.
So if the pool is imbalanced Depositing into GMXPool will not give optimal returns( resulting in fewer LP token), eventually leading to the loss of gain for the depositers affecting net APR

## Tools Used
Manual Review
## Recommendations
consider implementing check and if the pool is imablanced depositing(making levearge position) towards balancing the Index Token's weight will give optimal returns[extra LP tokens ] 
## <a id='H-02'></a>H-02. Positions may be liquidated due to incorrect implementation of Oracle logic            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-10-SteadeFi/blob/main/contracts/oracles/ChainlinkARBOracle.sol#L210

## Summary

Steadefi checks for historical data to make sure that last price update are within maximum delya allowed and in the range of maximum % deviation allowed.

But checking the historical data is incorrect according to the chainlink docs which can damage some serious logic with in the protcol

## Vulnerability Details

Vault calls [ChainlinkARBOracle.consult(token)]() to get the fair price from chainlink oracle

```solidity
File:

  function consult(address token) public view whenNotPaused returns (int256, uint8) {
    address _feed = feeds[token];

    if (_feed == address(0)) revert Errors.NoTokenPriceFeedAvailable();

    ChainlinkResponse memory chainlinkResponse = _getChainlinkResponse(_feed);
    ChainlinkResponse memory prevChainlinkResponse = _getPrevChainlinkResponse(_feed, chainlinkResponse.roundId);//@audit incorrect way to get historical data
    if (_chainlinkIsFrozen(chainlinkResponse, token)) revert Errors.FrozenTokenPriceFeed();
    if (_chainlinkIsBroken(chainlinkResponse, prevChainlinkResponse, token)) revert Errors.BrokenTokenPriceFeed();

 return (chainlinkResponse.answer, chainlinkResponse.decimals);
  }

```
https://github.com/Cyfrin/2023-10-SteadeFi/blob/main/contracts/oracles/ChainlinkARBOracle.sol#L62

which calls an interval function `_getPrevChainlinkResponse()` and try to fetch previous roundId price and other details

```solidity
  function _getPrevChainlinkResponse(address _feed, uint80 _currentRoundId) internal view returns (ChainlinkResponse memory) {
    ChainlinkResponse memory _prevChainlinkResponse;

    (
      uint80 _roundId,
      int256 _answer,
      /* uint256 _startedAt */,
      uint256 _timestamp,
      /* uint80 _answeredInRound */
    ) = AggregatorV3Interface(_feed).getRoundData(_currentRoundId - 1);

    _prevChainlinkResponse.roundId = _roundId;
    _prevChainlinkResponse.answer = _answer;
    _prevChainlinkResponse.timestamp = _timestamp;
    _prevChainlinkResponse.success = true;

    return _prevChainlinkResponse;
  }

```
https://github.com/Cyfrin/2023-10-SteadeFi/blob/main/contracts/oracles/ChainlinkARBOracle.sol#L210

But this is incorrect way of fetching historical data. 
chainlink docs say: `Oracles provide periodic data updates to the aggregators. Data feeds are updated in rounds. Rounds are identified by their roundId, which increases with each new round. This increase may not be monotonic. Knowing the roundId of a previous round allows contracts to consume historical data.

The examples in this document name the aggregator roundId as aggregatorRoundId to differentiate it from the proxy roundId.` [check here](https://docs.chain.link/data-feeds/historical-data#roundid-in-aggregator-aggregatorroundid)

so it is not mendatory that there will be valid data for currentRoundID-1.
if there is not data for currentRooundId-1 then `_badPriceDeviation(currChainlinkResponse,PrevResponse)` [check here](https://github.com/Cyfrin/2023-10-SteadeFi/blob/main/contracts/oracles/ChainlinkARBOracle.sol#L142) will return true. Hence vault won't able to get the price of token at some specific times
## Impact
1. In worse case keeper won't able to get the price of token so rebalancing , debt repay won't be possible leading to liquidation breaking the main important factor of protocol
2. Almost 70% of vault action is dependent on price of a token and not getting price will make them inactive affecting net APR 

## Tools Used
Manual Review
## Recommendations

As chainlink docs says. Increase in roundId may not be monotonic so loop through the previous roundID and fetch the previoous roundId data

pseudo  code
```solidity
 iterate (from roundId-1 to untill we get previous first data corressponding to roundID){
    if(data present for roundID){
        fetch the data and return
    }else{
        again iterate to get the data
    }
 }

```

		
# Medium Risk Findings

## <a id='M-01'></a>M-01. Not Choosing optimal path for swaps can result into loss of vaults            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-10-SteadeFi/blob/main/contracts/strategy/gmx/GMXWorker.sol#L114

https://github.com/Cyfrin/2023-10-SteadeFi/blob/main/contracts/strategy/gmx/GMXWorker.sol#L129

## Summary
Whenever swaps needed protocol use uniswap for optimal swaps. Which uses direct hardcoded direct path from input token to final token. This can result in greater slippage since direct path won't be always optimal path for swapping the tokens

## Vulnerability Details
```solodity
  struct SwapParams {
    // Address of token in
    address tokenIn;
    // Address of token out
    address tokenOut;
    // Amount of token in; in token decimals
    uint256 amountIn;
    // Amount of token out; in token decimals
    uint256 amountOut;
    // Slippage tolerance swap; e.g. 3 = 0.03%
    uint256 slippage;
    // Swap deadline timestamp
    uint256 deadline;
  }
```
This is swap params which does not include paths 
```solidity

File: GMXWorker.sol

  /**
    * @dev Swap exact amount of tokenIn for as many amount of tokenOut
    * @param self Vault store data
    * @param sp ISwap.SwapParams
    * @return amountOut Amount of tokens out in token decimals
  */
  function swapExactTokensForTokens(
    GMXTypes.Store storage self,
    ISwap.SwapParams memory sp
  ) external returns (uint256) {
    IERC20(sp.tokenIn).approve(address(self.swapRouter), sp.amountIn);

    return self.swapRouter.swapExactTokensForTokens(sp);
  }

  /**
    * @dev Swap as little tokenIn for exact amount of tokenOut
    * @param self Vault store data
    * @param sp ISwap.SwapParams
    * @return amountIn Amount of tokens in in token decimals
  */
  function swapTokensForExactTokens(
    GMXTypes.Store storage self,
    ISwap.SwapParams memory sp
  ) external returns (uint256) {
    IERC20(sp.tokenIn).approve(address(self.swapRouter), sp.amountIn);

    return self.swapRouter.swapTokensForExactTokens(sp);
  }

```
Direct Hardcoded path won't be optimal path always. There can be different optimal path for swaps but keeper will be loosing funds to make swap from this hardcoded paths

This will affect more to compounded tokens since for compounded tokens protocol won't able to select best pools 



## Impact
Keeper will loose funds in every funds for not choosing optimal paths. 
1. For Bluechip assets it can be optimal but there has been many instance where swaping becomes costly on direct paths
2. For Compounds, Keeper won't able to find optimal path and will lost funds for swapping directly.

## Tools Used

Manual Review

## Recommendations

Consider adding path param which can be used by keeper easily. 
Keeper can do offchain simulation and opt for best path to save slippage funds from swaps
## <a id='M-02'></a>M-02. Refund Fees will be sent to             

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-10-SteadeFi/blob/main/contracts/strategy/gmx/GMXDeposit.sol#L54

https://github.com/Cyfrin/2023-10-SteadeFi/blob/main/contracts/strategy/gmx/GMXTypes.sol#L170

https://github.com/Cyfrin/2023-10-SteadeFi/blob/main/contracts/strategy/gmx/GMXTypes.sol#L183

## Summary
After Deposits To GMX Pool extra funds is sent to refundee which is msg.sender. Which won't work for EIP4337 [Account's Abstraction] .

## Vulnerability Details

Analysing the current mass adoption of account abstraction we need to make sure the current logic is compatible for such users.

```solidity
File: GMXDeposit.sol
 /**
    * @notice @inheritdoc GMXVault
    * @param self GMXTypes.Store
    * @param isNative Boolean as to whether user is depositing native asset (e.g. ETH, AVAX, etc.)
  */
  function deposit(
    GMXTypes.Store storage self,
    GMXTypes.DepositParams memory dp,
    bool isNative
  ) external {
    // Sweep any tokenA/B in vault to the temporary trove for future compouding and to prevent
    // it from being considered as part of depositor's assets
    if (self.tokenA.balanceOf(address(this)) > 0) {
      self.tokenA.safeTransfer(self.trove, self.tokenA.balanceOf(address(this)));
    }
    if (self.tokenB.balanceOf(address(this)) > 0) {
      self.tokenB.safeTransfer(self.trove, self.tokenB.balanceOf(address(this)));
    }

    self.refundee = payable(msg.sender);

    [............]
  }
```
https://github.com/Cyfrin/2023-10-SteadeFi/blob/main/contracts/strategy/gmx/GMXDeposit.sol#L54

refundee has been set to msg.sender. But in the case user is using account's abstraction msg.sender will be bundler which executes the actual tx not the actual signer of tx.

So adding a receiver in deposit param could make this logic executable for such cases. User can make create useroperation with a receiver address and that address can be used as refundee in the current logic.

Even current logic will break if the user is blacklisted by the token so his funds will stuck in the protocol, Adding a redundee for withdrawl the token always work best
## Impact

1. Accounts abstraction is used by mass users and protocol is not compatible for such users.
2. User's funds can stuck in the prtocol if user is blacklisted  by the token

## Tools Used
Manual review
## Recommendations

Consider adding receiver param in deposit params as well as in withdraw params

# Low Risk Findings

## <a id='L-01'></a>L-01. Deposits and Withdrawal from Vault doesn't account deadline for transaction to complete            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-10-SteadeFi/blob/main/contracts/strategy/gmx/GMXTypes.sol#L183

https://github.com/Cyfrin/2023-10-SteadeFi/blob/main/contracts/strategy/gmx/GMXTypes.sol#L170

## Summary

Deposits and Withdrawal from Vault doesn't account deadline for transaction to complete. Transaction may not be optimal for user after certain deadline but protocol will execute them anyway

## Vulnerability Details
Protocol should provide their users with an option to limit the execution of their pending actions such as deposits and withdrawal. The most common solution is to include deadline timestamp and reverts if execution is after that timestamps like Uniswap.


Alice deposits 1000 index token to make delta position. She did some offchain computation and find that is she can get svTokens/GMXLP token from depositing before certain time she can avail some other benefits.
tx submitted in mempool and got executed after time she was expecting making a loss for Alice.

There can be muttiple cases where depositors and withdrawers would need deadline checks to make correct strategy for them which can't be implemented here

```solidity
File:contracts/strategy/gmx/GMXTypes.sol

  struct DepositParams {
    // Address of token depositing; can be tokenA, tokenB or lpToken
    address token;
    // Amount of token to deposit in token decimals
    uint256 amt;
    // Minimum amount of shares to receive in 1e18
    uint256 minSharesAmt;
    // Slippage tolerance for adding liquidity; e.g. 3 = 0.03%
    uint256 slippage;
    // Execution fee sent to GMX for adding liquidity
    uint256 executionFee;
  }

  struct WithdrawParams {
    // Amount of shares to burn in 1e18
    uint256 shareAmt;
    // Address of token to withdraw to; could be tokenA, tokenB or lpToken
    address token;
    // Minimum amount of token to receive in token decimals
    uint256 minWithdrawTokenAmt;
    // Slippage tolerance for removing liquidity; e.g. 3 = 0.03%
    uint256 slippage;
    // Execution fee sent to GMX for removing liquidity
    uint256 executionFee;
  }

```
Adding a deadline param can solve this issue. Please refer to uniswap and some other protocol which implement such checks with slippage

## Impact
Unexpected trade would get executed maliciously 
## Tools Used
Manual Review
## Recommendations
Add deadline parameter in deposits and withdrawal structs and revert if tx is executed after deadline


