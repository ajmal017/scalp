# Concurrent Scalp Algo

This python script is a working example to execute scalping trading algorithm for
[Alpaca API](https://alpaca.markets). This algorithm uses real time order updates
as well as minute level bar streaming from Polygon via Websockets.
One of the contributions of this example is to demonstrate how to handle
multiple stocks concurrently as independent routine using Python's asyncio.

The strategy holds positions up to 2 minute and exits positions quickly, so
you have to have more than $25k equity in your account due to the Pattern Day Trader rule,
to run this example.

## Dependency
This script needs latest [Alpaca Python SDK](https://github.com/alpacahq/alpaca-trade-api-python).
Please install it using pip

```sh
$ pip3 install alpaca-trade-api
```

or use pipenv using `Pipfile` in this directory.

```sh
$ pipenv install
```

## Usage

```sh
$ python main.py --lot=2000 TSLA FB AAPL
```

You can specify as many symbols as you want.  The script is designd to kick off while market
is open. Nothing would happen until 21 minutes from the market open as it relies on the
simple moving average as the buy signal.


## Strategy
The algorithm idea is to buy the stock upon the buy signal (MA20/price cross over in 1-minute bar) as
much as `lot` amount of dollar, then immediately sell the position at or above the entry price.
The buy signal is expremely simple, but what this strategy achieves is the quick reaction to
exit the position as soon as the buy order fills. There are reasonable chances that you can sell
the positions at the better prices than your entry within the short period of time. We send
limit order at the last trade or entry price whichever the higher to avoid unnecessary slippage.

The buy order is canceled after 2 minutes if it does not fill, assuming the signal is not
effective anymore. This could happen in a fast-moving market situation. Sells are left
indifinitely until it fills, but this may cause loss more than the accumulated profit depending
on the market situation. This is where you can improve the risk control beyond this example.

The buy signal is calculated as soon as a minute bar arrives, which typically happen about 4 seconds
after the top of every minute (this is Polygon's behavior for minute bar streaming).

This example liquidates all watching positions with market order at the end of market hours (03:55pm ET).


## Implementation
This example heavily relies on Python's asyncio. Although the thread is single, we handle
multiple symbols concurrently using this async loop.

We keep track of each symbol state in a separate `ScalpAlgo` class instance. That way,
everything stays simple without complex data struture and easy to read. The `main()`
function creates the algo instance for each symbol and creates streaming object
to listen the bar events. As soon as we receive a minute bar, we invoke event handler
for each symbol.

The main routine also starts a period check routine to do some work in background every 30 seconds.
In this background task, we check market state with the clock API and liquidate positions
before the market closes.

### Algo Instance and State Management
Each algo instance initializes its state by fetching day's bar data so far and position/order
from Alpaca API to synchronize, in case the script restarts after some trades. There are
four internal states and transitions as events happen.

- `TO_BUY`: no position, no order. Can transition to `BUY_SUBMITTED`
- `BUY_SUBMITTED`: buy order has been submitted. Can transition to `TO_BUY` or `TO_SELL`
- `TO_SELL`: buy is filled and holding position. Can transition to `SELL_SUBMITTED`
- `SELL_SUBMITTED`: sell order has been submitted. Can transition to `TO_SELL` or `TO_BUY`

### Event Handlers
`on_bar()` is an event handler for the bar data. Here we calculate signal that triggers
a buy order in the `TO_BUY` state. Once order is submitted, it goes to the `BUY_SUBMITTED`
state.

If order is filled, `on_order_update()` handler is called with `event=fill`. The state
transitions to `TO_SELL` and immediately submits a sell order, to transition to the
`SELL_SUBMITTED` state.

Orders may be canceled or rejected (caused by this script or you manually cancel them
from the dashboard). In these cases, the state transitions to `TO_BUY` (if not holding
a position) or `TO_SELL` (if holding a position) and wait for the next events.

`checkup()` method is the background periodic job to check several conditions, where
we cancel open orders and sends market sell order if there is an open position.

It exits once the market closes.

### Note
Each algo instance owns its child logger, prefixed by the symbol name. The console
log is also emitted to a file `console.log` under the same directory for your later review.

Again, the beautify of this code is that there is no multithread code but each
algo instance can focus on the bar/order/position data only for its own. It still
handles multiple symbols concurrently plus runs background periodic job in the
same async loop.

The trick to run additional async routine is as follows.

```py
    loop = stream.loop
    loop.run_until_complete(asyncio.gather(
        stream.subscribe(channels),
        periodic(),
    ))
    loop.close()
```

We use `asyncio.gather()` to run all bar handler, order update handler and periodic job
in one async loop indifinitely. You can kill it by `Ctrl+C`.