# Low Risk Findings

## <a id='L-01'></a>L-01. Chainlink price feeds can go stale            



### Summary

Chainlink price feeds can go stale, so there should be check to make sure they are not stale.

If the Chainlink feed goes stale, and the protocol is relying on it, then the protocol could be at risk.

### Vulnerability Detail

Stale price check is missing

### Impact
If the Chainlink feed goes stale, and the protocol is relying on it, then the protocol could be at risk.


### Code Snippet
```js
   function getUsdValue(address token, uint256 amount) public view returns (uint256) { 
        AggregatorV3Interface priceFeed = AggregatorV3Interface(s_priceFeeds[token]);
        (, int256 price,,,) = priceFeed.staleCheckLatestRoundData();
        // 1 ETH = $1000
        // The returned value from CL will be 1000 * 1e8
        return ((uint256(price) * ADDITIONAL_FEED_PRECISION) * amount) / PRECISION;
    }

```
### Tool used


### Recommendation

Check the price of the chainlink aggregator

```js
function getUsdValue(address token, uint256 amount) public view returns (uint256) {
    AggregatorV3Interface priceFeed = AggregatorV3Interface(s_priceFeeds[token]);
    (, int256 price,,,) = priceFeed.staleCheckLatestRoundData();

    // Check if the price retrieved is greater than zero
    require(price > 0, "Invalid price from Chainlink");

    // 1 ETH = $1000
    // The returned value from CL will be 1000 * 1e8
    return ((uint256(price) * ADDITIONAL_FEED_PRECISION) * amount) / PRECISION;
}
```


## <a id='L-03'></a>L-03. Chainlink oracle may return stale data and some functions can return wrong value.             





### Summary

 OracleLib.sol latestRoundData() is used but there is no check to validate if the return value contains fresh data. Proper checks should be applied to prevent the use of stale price data.

If the function getUsdValue does not properly verify and handle incorrect data related to the priceFeed, it could lead to incorrect results in functions that depend on it, such as getAccountCollateralValue, this, in turn, could affect the calculation of users' health factor and affect other critical functions, potentially put the project's funds at risk.

It is crucial that any function using external data, such as token prices obtained from oracles or third-party price feeds (such as Chainlink in this case), performs appropriate checks and error handling to ensure the accuracy and security of the calculations.


### Vulnerability Detail

```js
  function getUsdValue(address token, uint256 amount) public view returns (uint256) { 
        AggregatorV3Interface priceFeed = AggregatorV3Interface(s_priceFeeds[token]);
        (, int256 price,,,) = priceFeed.staleCheckLatestRoundData();
        // 1 ETH = $1000
        // The returned value from CL will be 1000 * 1e8
        return ((uint256(price) * ADDITIONAL_FEED_PRECISION) * amount) / PRECISION;
    }
    

```


This could lead to stale prices according to the Chainlink documentation:
https://docs.chain.link/data-feeds/price-feeds/historical-data



This, in turn, could affect the calculation of users' health factor and affect other critical functions, potentially put the project's funds at risk.

### Impact

-  If the function getUsdValue does not properly verify and handle incorrect data related to the priceFeed, it could lead to incorrect results in functions that depend on it, such as getAccountCollateralValue, this, in turn, could affect the calculation of users’ health factor and affect other critical functions, potentially put the project’s funds at risk.
### Code Snippet
https://github.com/Cyfrin/2023-07-foundry-defi-stablecoin/blob/d1c5501aa79320ca0aeaa73f47f0dbc88c7b77e2/src/DSCEngine.sol#L363C1-L363C1



### Tool used
Manual Review



### Recommendation
To ensure that the priceFeed is always correct, you can do that simply by adding some require statements in Oraclelib.sol.


```js
  function staleCheckLatestRoundData(AggregatorV3Interface priceFeed)
        public
        view
        returns (uint80, int256, uint256, uint256, uint80)
    {
        (uint80 roundId, int256 answer, uint256 startedAt, uint256 updatedAt, uint80 answeredInRound) =
            priceFeed.latestRoundData();
        uint256 secondsSince = block.timestamp - updatedAt;
        if (secondsSince > TIMEOUT) revert OracleLib__StalePrice();
        return (roundId, answer, startedAt, updatedAt, answeredInRound);
         require(answeredInRound >= roundId, "answer is stale");
        require(updatedAt > 0, "round is incomplete");
        require(answer > 0, "Invalid feed answer");
    }
```

