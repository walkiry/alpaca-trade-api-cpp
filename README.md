# Alpaca Trade API C++ Client

This document has the following sections:

- [Overview](#overview)
- [Contributing](#contributing)
- [Client Usage](#client-usage)
  - [Environment Variables](#environment-variables)
  - [Client Instantiation](#client-instantiation)
  - [Error Handling](#error-handling)
  - [Account API](#account-api)
  - [Account Configuration API](#account-configuration-api)
  - [Account Activities API](#account-activities-api)
  - [Orders API](#orders-api)
  - [Positions API](#positions-api)
  - [Assets API](#assets-api)
  - [Watchlist API](#watchlist-api)
  - [Calendar API](#calendar-api)
  - [Clock API](#clock-api)
- [Examples](#examples)
  - [Account Examples](#account-examples)
  - [Assets Examples](#assets-examples)
  - [Check Market Hours](#check-market-hours)
  - [Order Examples](#order-examples)
  - [Portfolio Examples](#porfolio-examples)


## Overview

`alpaca-trade-api-cpp` is a C++ library for the [Alpaca Commission Free Trading API](https://alpaca.markets). The official HTTP API documentation can be found at https://docs.alpaca.markets/.

## Contributing

For information on building, testing, and contributing to this repository, please see the [Contributor Guide](./CONTRIBUTING.md).

## Client Usage

### Environment Variables

The Alpaca SDK will check the environment for a number of variables which can be used rather to authenticate with and connect to the Alpaca API.

| Environment | Default Value | Description  |
| --- | --- | --- |
| APCA_API_KEY_ID | | Your API Key. |
| APCA_API_SECRET_KEY | | Your API Secret Key. |
| APCA_API_BASE_URL | paper-api.alpaca.markets | The endpoint for API calls. Note that the default is paper so you must specify this to switch to the live endpoint. |
| APCA_API_DATA_URL | data.alpaca.markets | The endpoint for the Data API. |

### Client Instantiation

To instantiate an instance of the API client, the main classes you'll need are:

- [`alpaca::Client`](./alpaca/client.h): The main API client class.
- [`alpaca::Environment](./alpaca/config.h): A helper class for parsing the required environment variables from the local environment.

Consider the following minimal example usage of these classes:

```cpp
#include <iostream>

#include "alpaca/client.h"
#include "alpaca/config.h"

int main(int argc, char* argv[]) {
  // Parse the required environment variables using the supplied helper utility
  auto env = alpaca::Environment();
  if (auto status = env.parse(); !status.ok()) {
    std::cout << "Error parsing config from environment: "
              << status.getMessage()
              << std::endl;
    return status.getCode();
  }

  // Instantiate an instance of the API client
  auto client = alpaca::Client(env)
}
```

### Error Handling

With few exceptions, most API client methods return a `std::pair` where the first item in the pair is an instance of `alpaca::Status`. The `alpaca::Status` class is used to represent the success or failure of the operation. The second item in the pair is the value that is requested, the response of API operation, etc.

```cpp
// Call the API
auto resp = client.getOrders();

// Check the status returned by the API client before using the response
if (auto status = resp.first; !status.ok()) {
  std::cerr << "The API operation failed: " << status.getMessage() << std::endl;
  return status.getCode();
}

// Now you can safely use the response object returned by the API
auto orders = resp.second;
for (const auto& order : orders) {
  std::cout << "Order ID: " << order.id << std::endl;
}
```

### Account API

The account API serves important information related to an account, including account status, funds available for trade, funds available for withdrawal, and various flags relevant to an account’s ability to trade. An account maybe be blocked for just for trades (the `trades_blocked` property of `alpaca::Account`) or for both trades and transfers (the `account_blocked` property of `alpaca::Account`) if Alpaca identifies the account to engaging in any suspicious activity. Also, in accordance with FINRA’s pattern day trading rule, an account may be flagged for pattern day trading (the `pattern_day_trader` property of `alpaca::Account`), which would inhibit an account from placing any further day-trades.

Consider the following example which exhibits how one can retrieve account information about the account which is currently authenticated.

```cpp
auto resp = client.getAccount();
if (auto status = resp.first; !status.ok()) {
  std::cerr << "Error calling API: " << status.getMessage() << std::endl;
  return status.getCode();
}

auto account = resp.second;
std::cout << "Account has buying power: " << account.buying_power << std::endl;
```

For more information the Account API, see the official API documentation: https://alpaca.markets/docs/api-documentation/api-v2/account/.

### Account Configuration API

The account configuration API provides custom configurations about your trading account settings. These configurations control various allow you to modify settings to suit your trading needs. For DTMC protection, see the documentation on [Day Trade Margin Call Protection](https://alpaca.markets/docs/trading-on-alpaca/user-protections/#day-trade-margin-call-dtmc-protection-at-alpaca).

Consider the following example which exhibits how one can check whether or not shorting is enabled and then conditionally enable shorting if it's not.

```cpp
auto resp = client.getAccountConfigurations();
if (auto status = resp.first; !status.ok()) {
  std::cerr << "Error calling API: " << status.getMessage() << std::endl;
  return status.getCode();
}

auto account_configurations = resp.second;
if (account_configurations.no_shorting) {
  std::cout << "Shorting is disabled for this account." << std::endl;

  auto update_resp = client.updateAccountConfigurations(
    false,
    account_configurations.dtbp_check,
    account_configurations.trade_confirm_email,
    account_configurations.suspend_trade
  );
  if (auto status = update_resp.first; !status.ok()) {
    std::cerr << "Error calling API: " << status.getMessage() << std::endl;
    return status.getCode();
  }
  std::cout << "Enabled shorting." << std::endl;
}
std::cout << "Shorting is enabled for this account." << std::endl;
```

For more information on the Account Configuration API, see the official API documentation: https://alpaca.markets/docs/api-documentation/api-v2/account-configuration/.

### Account Activities API

The account activities API provides access to a historical record of transaction activities that have impacted your account. Trade execution activities and non-trade activities, such as dividend payments, are both reported through this endpoint. At the time of this writing, the following are the types of activities that may be reported:

| Activity Type | Description |
| --- | --- |
| `FILL` |  Order fills (both partial and full fills) |
| `TRANS` | Cash transactions (both CSD and CSR) |
| `MISC` | Miscellaneous or rarely used activity types (All types except those in `TRANS`, `DIV`, or `FILL`) |
| `ACATC` | ACATS IN/OUT (Cash) |
| `ACATS` | ACATS IN/OUT (Securities) |
| `CSD` | Cash disbursement(+) |
| `CSR` | Cash receipt(-) |
| `DIV` | Dividends |
| `DIVCGL` | Dividend (capital gain long term) |
| `DIVCGS` | Dividend (capital gain short term) |
| `DIVFEE` | Dividend fee |
| `DIVFT` | Dividend adjusted (Foreign Tax Withheld) |
| `DIVNRA` | Dividend adjusted (NRA Withheld) |
| `DIVROC` | Dividend return of capital |
| `DIVTW` | Dividend adjusted (Tefra Withheld) |
| `DIVTXEX` | Dividend (tax exempt) |
| `INT` | Interest (credit/margin) |
| `INTNRA` | Interest adjusted (NRA Withheld) |
| `INTTW` | Interest adjusted (Tefra Withheld) |
| `JNL` | Journal entry |
| `JNLC` | Journal entry (cash) |
| `JNLS` | Journal entry (stock) |
| `MA` | Merger/Acquisition |
| `NC` | Name change |
| `OPASN` | Option assignment |
| `OPEXP` | Option expiration |
| `OPXRC` | Option exercise |
| `PTC` | Pass Thru Charge |
| `PTR` | Pass Thru Rebate |
| `REORG` | Reorg CA |
| `SC` | Symbol change |
| `SSO` | Stock spinoff |
| `SSP` | Stock split |

Consider the following example which exhibits how one can enumerate both trade and non-trade activity:

```cpp
auto resp = client.getAccountActivity();
if (auto status = resp.first; !status.ok()) {
  std::cerr << "Error calling API: " << status.getMessage() << std::endl;
  return status.getCode()
}

auto activities = resp.second;
for (const auto& activity : activities) {
  try {
    auto trade_activity = std::get<alpaca::TradeActivity>(activity);
    std::cout << "Trade Activity: " << std::endl;
    std::cout << "  Symbol = " << trade_activity.symbol << std::endl;
    std::cout << "  Side = " << trade_activity.side << std::endl;
    std::cout << "  Price = " << trade_activity.price << std::endl;
    std::cout << "  Quantity = " << trade_activity.qty << std::endl;
  } catch (const std::bad_variant_access&) {}
  try {
    auto non_trade_activity = std::get<alpaca::NonTradeActivity>(activity);
    std::cout << "Non-Trade Activity: " << std::endl;
    std::cout << "  Activity Type = " << non_trade_activity.activity_type << std::endl;
    std::cout << "  Symbol = " << non_trade_activity.symbol << std::endl;
    std::cout << "  Quantity = " << non_trade_activity.qty << std::endl;
  } catch (const std::bad_variant_access&) {}
}
```

For more information on the Account Activities API, see the official API documentation: https://alpaca.markets/docs/api-documentation/api-v2/account-activities/.

### Orders API

The Orders API allows a user to monitor, place and cancel their orders with Alpaca. Each order has a unique identifier provided by the client. This client-side unique order ID will be automatically generated by the system if not provided by the client, and will be returned as part of the order object along with the rest of the fields described below. Once an order is placed, it can be queried using the client-side order ID to check the status. Updates on open orders at Alpaca will also be sent over the streaming interface, which is the recommended method of maintaining order state.

For further details on order functionality, please see the [Trading On Alpaca - Orders](https://alpaca.markets/docs/trading-on-alpaca/orders/) page.

Consider the following example which exhibits buying 10 shares of NFLX, using the API to retieve the order by ID, and then using the API to cancel all open orders.

```cpp
// Submit an order to buy 10 shares of NFLX
auto submit_order_response = client.submitOrder(
  "NFLX",
  10,
  alpaca::OrderSide::Buy,
  alpaca::OrderType::Market,
  alpaca::OrderTimeInForce::Day
);
if (auto status = submit_order_response.first; !status.ok()) {
  std::cerr << "Error calling API: " << status.getMessage() << std::endl;
  return status.getCode();
}
auto order1 = submit_order_response.second;

// Use the API to retrieve the order we just placed.
auto get_order_response = client.getOrder(order.id);
if (auto status = get_order_response.first; !status.ok()) {
  std::cerr << "Error calling API: " << status.getMessage() << std::endl;
  return status.getCode();
}
auto order2 = get_order_response.second;

// Assert the client order IDs are the same for both order objects
assert(order1.client_order_id == order2.client_order_id);

// Cancel existing orders
auto cancel_orders_response = client.cancelOrders();
if (auto status = cancel_orders_response.first; !status.ok()) {
  std::cerr << "Error calling API: " << status.getMessage() << std::endl;
  return status.getCode();
}
```

For more information on the Orders API, see the official API documentation: https://alpaca.markets/docs/api-documentation/api-v2/orders/.

### Positions API

The positions API provides information about an account’s current open positions. The response will include information such as cost basis, shares traded, and market value, which will be updated live as price information is updated. Once a position is closed, it will no longer be queryable through this API.

Consider the following example which exhibits how to perform various API operations with the positions API.

```cpp
// Liquidate all existing positions
auto close_positions_response = client.closePositions()
if (auto status = close_positions_response.first; !status.ok()) {
  std::cerr << "Error calling API: " << status.getMessage() << std::endl;
  return status.getCode();
}

// Submit an order to buy 10 shares of NFLX
auto symbol = "NFLX";
auto submit_order_response = client.submitOrder(
  symbol,
  10,
  alpaca::OrderSide::Buy,
  alpaca::OrderType::Market,
  alpaca::OrderTimeInForce::Day
);
if (auto status = submit_order_response.first; !status.ok()) {
  std::cerr << "Error calling API: " << status.getMessage() << std::endl;
  return status.getCode();
}

// Get all open positions
auto get_positions_response = client.getPositions();
if (auto status = get_positions_response.first; !status.ok()) {
  std::cerr << "Error calling API: " << status.getMessage() << std::endl;
  return status.getCode();
}

// Ensure there is an open position for the stock we just purchased
auto found_symbol = false;
alpaca::Position found_position;
for (const auto& position : positions) {
  if (position.symbol == symbol) {
    found_symbol = true;
    found_position = position;
    break;
  }
}
assert(found_symbol);

// Directly retrieve the position for the stock we just purchased
auto get_position_response = client.getPosition(symbol);
if (auto status = get_position_response.first; !status.ok()) {
  std::cerr << "Error calling API: " << status.getMessage() << std::endl;
  return status.getCode();
}

// Ensure the asset ID is the same for the position we found while enumerating
// all positions as well as the position we retrieved by symbol
assert(found_position.asset_id, get_position_response.second.asset_id);
```

For more information on the Positions API, see the official API documentation: https://alpaca.markets/docs/api-documentation/api-v2/positions/.

### Assets API

The assets API serves as the master list of assets available for trade and data consumption from Alpaca. Assets are sorted by asset class, exchange and symbol. Some assets are only available for data consumption via Polygon, and are not tradable with Alpaca. These `alpaca::Asset` objects will be marked with the `tradable` property set to `false`.

Consider the following example which exhibits how to perform various API operations with the assets API.

```cpp
// Submit an order to buy 10 shares of NFLX
auto symbol = "NFLX";
auto submit_order_response = client.submitOrder(
  symbol,
  10,
  alpaca::OrderSide::Buy,
  alpaca::OrderType::Market,
  alpaca::OrderTimeInForce::Day
);
if (auto status = submit_order_response.first; !status.ok()) {
  std::cerr << "Error calling API: " << status.getMessage() << std::endl;
  return status.getCode();
}

// Get all assets
auto get_assets_response = client.getAssets();
if (auto status = get_assets_response.first; !status.ok()) {
  std::cerr << "Error calling API: " << status.getMessage() << std::endl;
  return status.getCode();
}

// Ensure there is an asset for the stock we just purchased
auto found_symbol = false;
alpaca::Asset found_asset;
for (const auto& asset : assets) {
  if (asset.symbol == symbol) {
    found_symbol = true;
    found_asset = asset;
    break;
  }
}
assert(found_asset);

// Directly retrieve the asset for the stock we just purchased
auto get_asset_response = client.getAsset(symbol);
if (auto status = get_asset_response.first; !status.ok()) {
  std::cerr << "Error calling API: " << status.getMessage() << std::endl;
  return status.getCode();
}

// Ensure the ID is the same for the asset we found while enumerating all
// asset as well as the asset we retrieved by symbol
assert(found_asset.id, get_asset_response.second.id);
```

For more information on the Assets API, see the official API documentation: https://alpaca.markets/docs/api-documentation/api-v2/assets/.

### Watchlist API

The watchlist API provides CRUD operation for the account’s watchlist. An account can have multiple watchlists and each is uniquely identified by id but can also be addressed by user-defined name. Each watchlist is an ordered list of assets.

Consider the following example which exhibits how to use the API to create, read, update, and delete a watchlist.

```cpp
// Create a watchlist
auto create_watchlist_response = client.createWatchlist("My Watchlist", {"GOOG"});
 if (auto status = create_watchlist_response.first; !status.ok()) {
  std::cerr << "Error calling API: " << status.getMessage() << std::endl;
  return status.getCode();
}
auto watchlist = create_watchlist_response.second;

// Read the watchlist
auto get_update_response = client.getWatchlist(watchlist.id);
 if (auto status = get_watchlist_response.first; !status.ok()) {
  std::cerr << "Error calling API: " << status.getMessage() << std::endl;
  return status.getCode();
}
auto watchlist = get_watchlist_response.second;

// Update the symbols in the watchlist
auto update_watchlist_response = client.updateWatchlist(
  watchlist.id,
  watchlist.name,
  {"GOOG", "FB"}
);
 if (auto status = update_watchlist_response.first; !status.ok()) {
  std::cerr << "Error calling API: " << status.getMessage() << std::endl;
  return status.getCode();
}
watchlist = update_watchlist_response.second;

// Add a symbol to the watchlist
auto add_symbol_response = client.addSymbolToWatchlist(watchlist.id, "AAPL");
 if (auto status = add_symbol_response.first; !status.ok()) {
  std::cerr << "Error calling API: " << status.getMessage() << std::endl;
  return status.getCode();
}
watchlist = add_symbol_response.second;

// Remove a symbol from the watchliist
auto remove_symbol_response = client.removeSymbolFromWatchlist(watchlist.id, "AAPL");
 if (auto status = remove_symbol_response.first; !status.ok()) {
  std::cerr << "Error calling API: " << status.getMessage() << std::endl;
  return status.getCode();
}
watchlist = remove_symbol_response.second;

// Delete the watchlist
auto delete_watchlist_response = client.deleteWatchlist(watchlist.id);
if (auto status = delete_watchlist_response.first; !status.ok()) {
  std::cerr << "Error calling API: " << status.getMessage() << std::endl;
  return status.getCode();
}
```

For more information on the Watchlist API, see the official API documentation: https://alpaca.markets/docs/api-documentation/api-v2/watchlist/.

### Calendar API

The calendar API serves the full list of market days from 1970 to 2029. It can also be queried by specifying a start and/or end time to narrow down the results. In addition to the dates, the response also contains the specific open and close times for the market days, taking into account early closures.

Consider the following example which exhibits how to find the open and close times for a date range.

```cpp
auto get_calendar_response = client.getCalendar("2018-01-02", "2018-01-09");
if (auto status = get_calendar_response.first; !status.ok()) {
  std::cerr << "Error calling API: " << status.getMessage() << std::endl;
  return status.getCode();
}
auto dates = get_calendar_response.second;
for (const auto& date : dates) {
  std::cout << "On " << date.date <<
            << ", the market opened at " << date.open
            << " and closed at " << date.close
            << "." << std::endl;
}
```

For more information on the Calendar API, see the official API documentation: https://alpaca.markets/docs/api-documentation/api-v2/calendar/.

### Clock API

The clock API serves the current market timestamp, whether or not the market is currently open, as well as the times of the next market open and close.

Consider the following example which exhibits using the API to determiine the time the market will next open:

```cpp
auto get_clock_response = client.getClock();
if (auto status = get_clock_response.first; !status.ok()) {
  std::cerr << "Error calling API: " << status.getMessage() << std::endl;
  return status.getCode();
}
auto clock = get_clock_response.second;
std::cout << "Next open: " << clock.next_open << std::endl;
```

For more information on the Clock API, see the official API documentation: https://alpaca.markets/docs/api-documentation/api-v2/clock/.

## Examples

### Account Examples

#### View Account Information

By sending a GET request to the `/v1/account` endpoint, you can see various information about your account, such as the amount of buying power available or whether or not it has a [PDT flag](https://support.alpaca.markets/hc/en-us/articles/360012203032-Pattern-Day-Trader).

```cpp
#include <iostream>

#include "alpaca/alpaca.h"

int main(int argc, char* argv[]) {
  auto env = alpaca::Environment();
  if (auto status = env.parse(); !status.ok()) {
    std::cerr << "Error parsing config from environment: " << status.getMessage() << std::endl;
    return status.getCode();
  }
  auto client = alpaca::Client(env);

  // Get our account information.
  auto account_response = client.getAccount();
  if (auto status = account_response.first; !status.ok()) {
    std::cerr << "Error getting account information: " << status.getMessage() << std::endl;
    return status.getCode();
  }
  auto account = account_response.second;

  // Check if our account is restricted from trading.
  if (account.trading_blocked) {
    std::cout << "Account is currently restricted from trading." << std::endl;
  }

  // Check how much money we can use to open new positions.
  std::cout << account.buying_power << " is available as buying power." << std::endl;

  return 0;
}
```

#### View Gain/Loss of Portfolio

You can use the information from the account endpoint to do things like calculating the daily profit or loss of your account.

```cpp
#include <iostream>

#include "alpaca/alpaca.h"

int main(int argc, char* argv[]) {
  auto env = alpaca::Environment();
  if (auto status = env.parse(); !status.ok()) {
    std::cerr << "Error parsing config from environment: " << status.getMessage() << std::endl;
    return status.getCode();
  }
  auto client = alpaca::Client(env);

  // Get account info
  auto account_response = client.getAccount();
  if (auto status = account_response.first; !status.ok()) {
    std::cerr << "Error getting account information: " << status.getMessage() << std::endl;
    return status.getCode();
  }
  auto account = account_response.second;

  // Check our current balance vs. our balance at the last market close
  std::cout << "Equity (cash + long_market_value + short_market_value): " << account.equity;
  std::cout << "Equity as of previous trading day: " << account.last_equity;

  return 0;
}
```

### Assets Examples

#### Get a List of Assets

If you send a GET request to our `/v1/assets` endpoint, you’ll receive a list of US equities.

```cpp
#include <iostream>

#include "alpaca/alpaca.h"

int main(int argc, char* argv[]) {
  auto env = alpaca::Environment();
  if (auto status = env.parse(); !status.ok()) {
    std::cerr << "Error parsing config from environment: " << status.getMessage() << std::endl;
    return status.getCode();
  }
  auto client = alpaca::Client(env);

  // Get a list of all active assets.
  auto assets_response = client.getAssets();
  if (auto status = assets_response.first; !status.ok()) {
    std::cerr << "Error getting assets information: " << status.getMessage() << std::endl;
    return status.getCode();
  }
  auto assets = assets_response.second;

  // Filter the assets down to just those on NASDAQ.
  std::vector<alpaca::Asset> nasdaq_assets;
  for (const auto& asset : assets) {
    if (asset.exchange == "NASDAQ") {
      nasdaq_assets.push_back(asset);
    }
  }

  std::cout << "Found " << nasdaq_assets.size() << " assets being traded on the NASDAQ exchange" << std::endl;

  return 0;
}
```

#### See If a Particular Asset is Tradable on Alpaca

By sending a symbol along with our request, we can get the information about just one asset. This is useful if we just want to make sure that a particular asset is tradable before we attempt to buy it.

```cpp
#include <iostream>

#include "alpaca/alpaca.h"

int main(int argc, char* argv[]) {
  auto env = alpaca::Environment();
  if (auto status = env.parse(); !status.ok()) {
    std::cerr << "Error parsing config from environment: " << status.getMessage() << std::endl;
    return status.getCode();
  }
  auto client = alpaca::Client(env);

  // Check if AAPL is tradable on the Alpaca platform.
  auto asset_response = client.getAsset("AAPL");
  if (auto status = asset_response.first; !status.ok()) {
    std::cerr << "Error getting asset information: " << status.getMessage() << std::endl;
    return status.getCode();
  }
  auto asset = asset_response.second;

  if (asset.tradable) {
    std::cout << "We can trade AAPL." << std::endl;
  } else {
    std::cout << "We can not trade AAPL." << std::endl;
  }

  return 0;
}
```

### Check Market Hours

#### See if the Market is Open

With GET requests to the `/v1/calendar` and `/v1/clock` endpoints, you can check if the market is open now, or view what times the market will be open or closed on a particular date.

```cpp
#include <iostream>

#include "alpaca/alpaca.h"

int main(int argc, char* argv[]) {
  auto env = alpaca::Environment();
  if (auto status = env.parse(); !status.ok()) {
    std::cerr << "Error parsing config from environment: " << status.getMessage() << std::endl;
    return status.getCode();
  }
  auto client = alpaca::Client(env);

  // Check if the market is open now.
  auto clock_response = client.getClock();
  if (auto status = clock_response.first; !status.ok()) {
    std::cerr << "Error getting clock information: " << status.getMessage() << std::endl;
    return status.getCode();
  }
  auto clock = clock_response.second;

  if (clock.is_open) {
    std::cout << "The market is open." << std::endl;
  } else {
    std::cout << "The market is closed." << std::endl;
  }

  // Check when the market was open on Dec. 1, 2018
  auto date = "2018-12-01";
  auto calendar_response = client.getCalendar(date, date);
  if (auto status = calendar_response.first; !status.ok()) {
    std::cerr << "Error getting calendar information: " << status.getMessage() << std::endl;
    return status.getCode();
  }
  auto days = calendar_response.second;
  if (auto size = days.size(); size != 1) {
    std::cerr << "Expected to receive 1 day result but got " << size << "instead." << std::endl;
  }
  auto day = days.front();
  std::cout << "The market opened at " << day.open << " and closed at " << day.close << " on " << day.date << "."
            << std::endl;
  return 0;
}
```

### Order Examples

These are examples of some of the things you can do with order objects through the Alpaca API. For additional help understanding different types of orders and how they behave once they’re placed, please see [this page](https://docs.alpaca.markets/orders/).

#### Place New Orders

Orders can be placed with a POST request to the `/v1/orders` endpoint.

```cpp
#include <iostream>

#include "alpaca/alpaca.h"

int main(int argc, char* argv[]) {
  auto env = alpaca::Environment();
  if (auto status = env.parse(); !status.ok()) {
    std::cerr << "Error parsing config from environment: " << status.getMessage() << std::endl;
    return status.getCode();
  }
  auto client = alpaca::Client(env);

  // Submit a market order to buy 1 share of Apple at market price
  auto buy_response =
      client.submitOrder("AAPL", 1, alpaca::OrderSide::Buy, alpaca::OrderType::Market, alpaca::OrderTimeInForce::Day);
  if (auto status = buy_response.first; !status.ok()) {
    std::cerr << "Error submitting purchase order: " << status.getMessage() << std::endl;
    return status.getCode();
  }

  // Submit a limit order to attempt to sell 1 share of AMD at a particular
  // price ($20.50) when the market opens
  auto sell_response = client.submitOrder(
      "AMD", 1, alpaca::OrderSide::Sell, alpaca::OrderType::Limit, alpaca::OrderTimeInForce::OPG, "20.50");
  if (auto status = sell_response.first; !status.ok()) {
    std::cerr << "Error submitting sell order: " << status.getMessage() << std::endl;
    return status.getCode();
  }

  return 0;
}
```

#### Use Client Order IDs

Client Order IDs can be used to organize and track specific orders in your client program.

```cpp
#include <iostream>

#include "alpaca/alpaca.h"

int main(int argc, char* argv[]) {
  auto env = alpaca::Environment();
  if (auto status = env.parse(); !status.ok()) {
    std::cerr << "Error parsing config from environment: " << status.getMessage() << std::endl;
    return status.getCode();
  }
  auto client = alpaca::Client(env);

  // Submit a market order to buy 1 share of Apple at market price
  auto buy_response =
      client.submitOrder("AAPL", 1, alpaca::OrderSide::Buy, alpaca::OrderType::Market, alpaca::OrderTimeInForce::Day, "", "", false, "my_first_order");
  if (auto status = buy_response.first; !status.ok()) {
    std::cerr << "Error submitting purchase order: " << status.getMessage() << std::endl;
    return status.getCode();
  }

  // Get order by it's client order ID
  auto get_order_response = client.getOrderByClientOrderID("my_first_order");
  if (auto status = get_order_response.first; !status.ok()) {
    std::cerr << "Error getting order: " << status.getMessage() << std::endl;
    return status.getCode();
  }
  auto order = get_order_response.second;

  std::cout << "Got order by client order ID: " << order.id;

  return 0;
}
```

#### Get a List of Existing Orders
If you’d like to see a list of your existing orders, you can send a get request to the `/v1/orders` endpoint.

```cpp
#include <iostream>

#include "alpaca/alpaca.h"

int main(int argc, char* argv[]) {
  auto env = alpaca::Environment();
  if (auto status = env.parse(); !status.ok()) {
    std::cerr << "Error parsing config from environment: " << status.getMessage() << std::endl;
    return status.getCode();
  }
  auto client = alpaca::Client(env);

  // Get the last 100 of our closed orders
  auto list_orders_response = client.getOrders(alpaca::ActionStatus::Closed, 100);
  if (auto status = list_orders_response.first; !status.ok()) {
    std::cerr << "Error listing orders: " << status.getMessage() << std::endl;
    return status.getCode();
  }
  auto orders = list_orders_response.second;

  // Get only the closed orders for a particular stock
  std::vector<alpaca::Order> aapl_orders;
  for (const auto& order : orders) {
    if (order.symbol == "AAPL") {
      aapl_orders.push_back(order);
    }
  }
  std::cout << "Found " << aapl_orders.size() << " closed orders for AAPL." << std::endl;

  return 0;
}
```

### Portfolio Examples

#### View Open Positions in Your Portfolio

You can view the positions in your portfolio by making a GET request to the `/v1/positions` endpoint. If you specify a symbol, you’ll see only your position for the associated stock.

```cpp
#include <iostream>

#include "alpaca/alpaca.h"

int main(int argc, char* argv[]) {
  auto env = alpaca::Environment();
  if (auto status = env.parse(); !status.ok()) {
    std::cerr << "Error parsing config from environment: " << status.getMessage() << std::endl;
    return status.getCode();
  }
  auto client = alpaca::Client(env);

  // Get our position in AAPL.
  auto get_position_response = client.getPosition("AAPL");
  if (auto status = get_position_response.first; status.ok()) {
    auto position = get_position_response.second;
    std::cout << "Apple position: " << position.qty << " shares." << std::endl;
  } else {
    std::cout << "No AAPL position.";
  }

  // Get a list of all of our positions.
  auto get_positions_response = client.getPositions();
  if (auto status = get_positions_response.first; !status.ok()) {
    std::cerr << "Error getting positions information: " << status.getMessage() << std::endl;
    return status.getCode();
  }
  auto positions = get_positions_response.second;
  for (const auto& position : positions) {
    std::cout << position.qty << " shares in " << position.symbol << std::endl;
  }

  return 0;
}
```