# Trading Engine (Express + In-Memory Matching)

A simplified trading engine backend with a small web UI.

It supports:
- Limit order placement (buy/sell)
- Price-time priority matching
- Partial and full fills
- Order cancellation (open/partially filled only)
- Pair-specific order books
- Executed trade history
- Terminal logging for all engine events

## Tech Stack

- Node.js
- Express.js
- UUID
- In-memory storage (no database)

## Supported Pairs

This project is configured for **exactly 2 pairs**:
- `BTC/USD`
- `KGEN/USDT`

The pair list is defined in `config/marketConfig.js` and startup enforces that exactly two pairs are configured.

## Project Structure

```text
trading_engine/
├── app.js
├── config/
│   └── marketConfig.js
├── controllers/
│   └── orderController.js
├── engine/
│   └── matchingEngine.js
├── models/
│   ├── Order.js
│   └── Trade.js
├── public/
│   ├── app.js
│   ├── index.html
│   └── style.css
├── routes/
│   └── orderRoutes.js
├── services/
│   └── orderBookService.js
└── utils/
    └── logger.js
```

## Run Locally

```bash
npm install
npm start
```

App starts on:
- UI: `http://localhost:3000`
- API: `http://localhost:3000`

## Matching Rules

### Buy order
- Matches against the **lowest sell price first**
- Eligible when: `sell.price <= buy.price`
- FIFO within same price level

### Sell order
- Matches against the **highest buy price first**
- Eligible when: `buy.price >= sell.price`
- FIFO within same price level

### Fill behavior
- Supports partial fills
- Updates `remaining` after every trade
- Marks status as `open`, `partially_filled`, `filled`, or `cancelled`
- Removes fully filled resting orders from the order book

## API Endpoints

### 1) Place order
`POST /order`

Body:

```json
{
  "pair": "BTC/USD",
  "type": "buy",
  "price": 100,
  "quantity": 5
}
```

### 2) Cancel order
`DELETE /order/:id`

### 3) Get orderbook
`GET /orderbook`

Optional filter:
- `GET /orderbook?pair=BTC%2FUSD`

### 4) Get trades
`GET /trades`

Optional filter:
- `GET /trades?pair=KGEN%2FUSDT`

### 5) Get configured pairs
`GET /pairs`

## Sample cURL

```bash
# Get pairs
curl -s http://localhost:3000/pairs

# Place buy order
curl -s -X POST http://localhost:3000/order \
  -H "Content-Type: application/json" \
  -d '{"pair":"BTC/USD","type":"buy","price":101,"quantity":5}'

# Place sell order that can match
curl -s -X POST http://localhost:3000/order \
  -H "Content-Type: application/json" \
  -d '{"pair":"BTC/USD","type":"sell","price":100,"quantity":2}'

# Pair-filtered orderbook
curl -s 'http://localhost:3000/orderbook?pair=BTC%2FUSD'

# Pair-filtered trades
curl -s 'http://localhost:3000/trades?pair=BTC%2FUSD'

# Cancel an order
curl -s -X DELETE http://localhost:3000/order/<ORDER_ID>

```

## UI Overview

Open `http://localhost:3000`.

Features:
- Pair selector (2 pairs)
- Place buy/sell limit orders
- Live buy/sell orderbook tables
- Open orders with cancel action
- Executed trades list
- Trading-terminal style layout

## Validation Rules

Order requests validate:
- `pair` must be one of configured pairs
- `type` must be `buy` or `sell`
- `price` must be a positive number
- `quantity` must be a positive number

## Logging

The app logs:
- Incoming HTTP requests
- Order receive/process lifecycle
- Trade executions
- Cancellation success/failure

Logs are available:
- Terminal stdout only (VS Code terminal)

## Notes

- Data is in memory only and resets on server restart.
- No authentication is included.
- This is intentionally simplified for matching-engine behavior clarity.
