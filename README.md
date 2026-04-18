# Trading Engine

Simplified in-memory trading engine with:
- Express backend
- Matching engine (price-time priority)
- Trading dashboard frontend

No database is used. All state resets when server restarts.

## Features

- Place limit buy/sell orders
- Match orders using price priority, then FIFO time priority
- Partial fills and full fills
- Cancel open or partially-filled orders
- Separate order books per trading pair
- Executed trade history
- Terminal logs for order/trade lifecycle

## Supported Pairs

Exactly 2 pairs are configured:
- `BTC/USD`
- `KGEN/USDT`

Source: `config/marketConfig.js`

## Tech Stack

- Node.js
- Express.js
- UUID
- Vanilla JS frontend (modular components)

## Architecture

![Trading Engine Architecture](docs/architecture.png)

### Data Flow

1. **Order Placement**: Frontend sends POST /order → Backend validates → Matching Engine processes → Trade created if matched
2. **Order Book**: Frontend polls GET /orderbook → Returns live bid/ask orders per pair
3. **Trades**: Frontend polls GET /trades → Returns executed trades per pair
4. **Cancellation**: Frontend sends DELETE /order/:id → Backend updates order status

### Storage

All data stored in-memory:
- `orderBook`: Maps pair → {bid orders, ask orders}
- `trades`: Array of executed trades
- `ordersById`: Historical order lookup

Data is **NOT persisted**. Server restart clears all state.

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
├── routes/
│   └── orderRoutes.js
├── services/
│   └── orderBookService.js
├── utils/
│   └── logger.js
├── public/
│   ├── index.html
│   ├── style.css
│   ├── app.js
│   ├── api.js
│   └── components/
│       ├── orderForm.js
│       ├── orderBook.js
│       ├── trades.js
│       ├── openOrders.js
│       └── toasts.js
└── README.md
```

## Run Locally

```bash
npm install
npm start
```

Default server:
- `http://localhost:3000`

Health check:

```bash
curl -s http://localhost:3000/health
```

Expected:

```json
{"status":"ok"}
```

## API Endpoints

### `POST /order`
Place order.

Request body:

```json
{
  "pair": "BTC/USD",
  "type": "buy",
  "price": 100,
  "quantity": 5
}
```

### `DELETE /order/:id`
Cancel order (only `open` or `partially_filled`).

### `GET /orderbook`
Get order book.

Optional pair filter:
- `GET /orderbook?pair=BTC%2FUSD`

### `GET /trades`
Get executed trades.

Optional pair filter:
- `GET /trades?pair=KGEN%2FUSDT`

### `GET /pairs`
Get configured pairs.

## Matching Logic

### Buy incoming order
- Matches against lowest sell first
- Match condition: `sell.price <= buy.price`
- FIFO for same sell price level

### Sell incoming order
- Matches against highest buy first
- Match condition: `buy.price >= sell.price`
- FIFO for same buy price level

### Fill handling
- Partial fills supported
- `remaining` updates after each trade
- Status transitions: `open` → `partially_filled` → `filled`
- Filled resting orders removed from order book

## Frontend Notes

Open `http://localhost:3000`.

Dashboard includes:
- Place Order panel (pair, buy/sell tabs, price, quantity)
- Order book (bids/asks)
- Trades table
- Open orders with cancel button
- Loading, empty, and toast states

### API URL in UI

Top bar has `API URL` input. Default is current origin (typically `http://localhost:3000`).

If backend runs elsewhere (example `http://localhost:5000`):
1. Enter that URL in `API URL`
2. Click `Save`

## Sample cURL

```bash
# Pairs
curl -s http://localhost:3000/pairs

# Place buy order
curl -s -X POST http://localhost:3000/order \
  -H "Content-Type: application/json" \
  -d '{"pair":"BTC/USD","type":"buy","price":101,"quantity":5}'

# Place sell order
curl -s -X POST http://localhost:3000/order \
  -H "Content-Type: application/json" \
  -d '{"pair":"BTC/USD","type":"sell","price":100,"quantity":2}'

# Orderbook for pair
curl -s 'http://localhost:3000/orderbook?pair=BTC%2FUSD'

# Trades for pair
curl -s 'http://localhost:3000/trades?pair=BTC%2FUSD'

# Cancel order
curl -s -X DELETE http://localhost:3000/order/<ORDER_ID>
```

## Logging

Logs print to backend terminal (`npm start` terminal):
- Server start
- HTTP order/cancel requests
- Order received/processed
- Trade executed
- Cancel success/failure

## Troubleshooting

### Error: `EADDRINUSE: port 3000`
Another process is already using port 3000.

```bash
pkill -f "node app.js" || true
npm start
```

Or inspect port:

```bash
lsof -nP -iTCP:3000 -sTCP:LISTEN
```

### Site not reachable
- Ensure backend is running (`npm start`)
- Use `http://localhost:3000` (not https)
- Verify health endpoint

### Frontend shows fetch errors
- Check backend health
- Ensure API URL in UI points to running backend
- Check CORS/network restrictions if using different origin

## Limitations

- In-memory only; no persistence
- Not production-hardened (no auth/rate limiting)
- `ordersById` keeps historical orders in memory until restart

## Future Improvements

- Add user authentication (JWT-based)
- Associate orders with user accounts
- Persist order book and trades using a database
- Real-time updates using WebSockets
