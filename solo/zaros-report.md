### H-1 Incorrect Logic Used in Determining `ctx.isFillPriceValid` in `SettlementBranch::fillOffchainOrders` Function

## Description:

Incorrect logic was applied while determining if `ctx.isFillPriceValid` is really valid. When placing a buy (long position), the fill price should be less than the target price (`fillPrice < targetPrice`). Conversely, when placing a sell (short position), the fill price should be greater than the target price (`fillPrice > targetPrice`).

```Solidity
function fillOffchainOrders(
    uint128 marketId,
    OffchainOrder.Data[] calldata offchainOrders,
    bytes calldata priceData
) external onlyOffchainOrdersKeeper(marketId) {
    // working data
    FillOffchainOrders_Context memory ctx;

    // omitted for brevity...

    ctx.isFillPriceValid = (ctx.isBuyOrder && ctx.offchainOrder.targetPrice <=ctx.fillPriceX18.intoUint256())
        || (!ctx.isBuyOrder && ctx.offchainOrder.targetPrice >= ctx.fillPriceX18.intoUint256());

    // we don't revert here because we want to continue filling other orders.
    if (!ctx.isFillPriceValid) {
        continue;
    }

    // account state updates start here

    // increase the trading account nonce if the order's flag is true.
    if (ctx.offchainOrder.shouldIncreaseNonce) {
        unchecked {
            tradingAccount.nonce++;
        }
    }

    // mark the offchain order as filled.
    // we store the struct hash to be marked as filled.
    tradingAccount.hasOffchainOrderBeenFilled[ctx.structHash] = true;

    // fill the offchain order.
    _fillOrder(
        ctx.offchainOrder.tradingAccountId,
        marketId,
        SettlementConfiguration.OFFCHAIN_ORDERS_CONFIGURATION_ID,
        sd59x18(ctx.offchainOrder.sizeDelta),
        ctx.fillPriceX18
    );
}
```

## Impact:

The incorrect logic for validating the fill price can lead to unintended trades being executed. This could result in the following issues:

* **Financial Losses:** Traders may suffer financial losses due to trades being executed at unfavorable prices.
* **Market Manipulation:** Malicious actors could exploit this logic to manipulate the market, taking advantage of incorrectly validated orders.
* **Loss of Trust:** The integrity of the trading platform could be compromised, leading to a loss of trust from users and a potential decrease in platform usage.

**Proof of Concept:**

A proof of concept would involve creating a scenario where a buy order is placed with a fill price higher than the target price, or a sell order is placed with a fill price lower than the target price.

From the code Natspec, "if the order increases the trading account's position (buy order), the fill price must be less than or equal to the target price. If it decreases the trading account's position (sell order), the fill price must be greater than or equal to the target price."

Judging by this, we expect the logic to be:

* Buy: `fillPrice <= targetPrice`
* Sell: `fillPrice >= targetPrice`

From the current logic:

```solidity
ctx.isFillPriceValid = (ctx.isBuyOrder && ctx.offchainOrder.targetPrice <= ctx.fillPriceX18.intoUint256())
    || (!ctx.isBuyOrder && ctx.offchainOrder.targetPrice >= ctx.fillPriceX18.intoUint256());
```

There will always be an issue because `ctx.isFillPriceValid` will always return false. For example:

* Buy: `10.5 <= 11.5` (target price for buy is 10.5, fill price for buy is 11.5)
* Sell: `9.5 >= 17.5` (target price for sell is 9.5, fill price for sell is 17.5)

Due to the incorrect logic, these orders would incorrectly pass validation and be executed.

## Tools Used

Manual Audit

## Recommended Mitigation:

To address this issue, correct the logic for validating the fill price in the `fillOffchainOrders` function. Ensure that for buy orders, the fill price is less than the target price, and for sell orders, the fill price is greater than the target price.


### L-1 The Use of `payable` in `TradingAccount::createTradingAccountAndMulticall` Could Lead to Funds Getting Stuck in the Protocol

Description:

The function TradingAccount::createTradingAccountAndMulticall is marked as payable, potentially leading to users mistakenly depositing Ether into the protocol when creating an account with the Multicall functionality. Since msg.value is ignored, any Ether sent to this function will be permanently locked in the contract.
function createTradingAccountAndMulticall(
        bytes[] calldata data,
        bytes memory referralCode,
        bool isCustomReferralCode
    )
        external
@>>  payable // ignored msg.value
        virtual
        returns (bytes[] memory results)
    {
        uint128 tradingAccountId = createTradingAccount(referralCode, isCustomReferralCode);

        results = new bytes[](data.length);
        for (uint256 i; i < data.length; i++) {
            bytes memory dataWithAccountId = bytes.concat(data[i][0:4], abi.encode(tradingAccountId), data[i][4:]);
            (bool success, bytes memory result) = address(this).delegatecall(dataWithAccountId);
            
            if (!success) {
                uint256 len = result.length;
                assembly {
                    revert(add(result, 0x20), len)
                }
            }

            results[i] = result;
        }
    }
Impact:

If users mistakenly send Ether when calling this function, the Ether will be irretrievably locked in the contract, leading to:

Loss of Funds: Users may unintentionally lose their Ether by sending it to a function that does not process it.

User Frustration: Users who mistakenly send Ether may become frustrated with the platform, potentially leading to a loss of trust and a decrease in user retention.

Proof of Concept:

A proof of concept would involve calling the createTradingAccountAndMulticall function and sending Ether along with the call. The Ether sent will not be processed and will be stuck in the contract.

Recommended Mitigation:

Remove the payable modifier from the createTradingAccountAndMulticall function to address this issue. This will prevent users from mistakenly sending Ether to the function.

Additionally, you can add a withdraw function to enable users who have deposited Ether to withdraw.

function createTradingAccountAndMulticall(
        bytes[] calldata data,
        bytes memory referralCode,
        bool isCustomReferralCode
    )
        external
        virtual
        returns (bytes[] memory results)
    {
        uint128 tradingAccountId = createTradingAccount(referralCode, isCustomReferralCode);

        results = new bytes[](data.length);
        for (uint256 i; i < data.length; i++) {
            bytes memory dataWithAccountId = bytes.concat(data[i][0:4], abi.encode(tradingAccountId), data[i][4:]);
            (bool success, bytes memory result) = address(this).delegatecall(dataWithAccountId);
            // @audit-info is dataWithAccountId trusted?? use of delegatecall can be risky,
            // since the func is payable n theres a delegatecall its possible to loose funds
            if (!success) {
                uint256 len = result.length;
                assembly {
                    revert(add(result, 0x20), len)
                }
            }

            results[i] = result;
        }
    }
By implementing this change, the risk of Ether being mistakenly sent and stuck in the contract will be eliminated, thereby protecting user funds and maintaining the integrity of the protocol.

```solidity
ctx.isFillPriceValid = (ctx.isBuyOrder && ctx.offchainOrder.targetPrice >= ctx.fillPriceX18.intoUint256())
    || (!ctx.isBuyOrder && ctx.offchainOrder.targetPrice <= ctx.fillPriceX18.intoUint256());
```

By implementing this change, only orders with valid fill prices will be executed, thereby preventing potential financial losses and maintaining the integrity of the trading platform.
