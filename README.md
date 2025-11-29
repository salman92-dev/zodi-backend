# ZOODI Token Eligibility Backend

A Node.js/Express backend service that tracks Solana wallet eligibility based on ZOODI token holdings and Raydium CLMM LP positions. Features fast eligibility checks with RPC pooling, cron-based batch processing, and MongoDB persistence.

---

## Features

- **Fast Eligibility Checks**: Detects ZOODI token balances and ZOODI-WSOL CLMM LP positions
- **Multi-RPC Failover**: Rotates between multiple RPC endpoints with exponential backoff on rate-limits
- **Paginated Position Queries**: Uses `getProgramAccountsV2` pagination to handle large datasets without deprioritization
- **Concurrent Request Handling**: Chunked account-info fetches (configurable batch sizes) to avoid RPC throttling
- **Daily Cron Batch Processing**: Automatically scans and updates all users' eligibility daily
- **Database Caching**: Quick responses via MongoDB snapshots with background refresh
- **CORS Support**: Configured to accept requests from frontend (default: `localhost:5173`)
- **Fallback Parsing**: Handles both Token and Token-2022 account programs with automatic raw-account decoding fallback

---

## Architecture

### Core Services

- **eligibilityService**: Main eligibility detection logic
  - `checkEligibility(walletAddress)` — Scans token balance and LP positions, updates DB
  - `getAllLPPositions(walletAddress)` — Fast-path (memcmp filter) + slow-path (candidate PDA scan)
  - `getTokenBalance(walletAddress, mint)` — Fetches ZOODI balance with fallback parsing

- **cronService**: Scheduled daily batch processing
  - Runs daily at 00:00 UTC
  - Configurable batch size and inter-batch delays
  - Exponential backoff on rate-limit errors

- **rpcPool**: Multi-endpoint RPC management
  - Automatic rotation on 429 / rate-limit errors
  - Exponential backoff with configurable attempts and base delay
  - Endpoint logging for observability

### Data Flow

```
Frontend (localhost:5173)
    ↓
POST /api/check-eligibility (walletAddress)
    ↓
Express Controller
    ↓
eligibilityService.checkEligibility()
    ├─ Fetch parsed token accounts (Token + Token-2022 programs)
    ├─ Fast-path: getProgramAccounts → memcmp filter by pool ID
    ├─ Fallback: getProgramAccountsV2 (paginated) if deprioritized
    ├─ Fetch position account data & parse liquidity
    ├─ Store/update user record in MongoDB
    └─ Return eligibility status + LP details

Cron Job (daily 00:00)
    ├─ Fetch all users from DB (in batches of N)
    ├─ Call checkEligibility for each batch
    └─ Update days_completed, time_remaining, claim_eligible
```

---

## API Endpoints

### Check Eligibility
```
POST /api/check-eligibility
Content-Type: application/json

{
  "walletAddress": "HAzWB9uik5fUmA6c9ErcMnSrHDDFuR7L7YTHNf2nAqNu"
}

Query Params:
  ?force=true    — Force fresh check (bypass cache)

Response:
{
  "eligibilityStatus": "Eligible",
  "eligibilityType": ["LP", "Holding"],
  "lpAmount": 434.683,
  "lpPositionCount": 3,
  "tokenBalance": 6278.706,
  "daysCompleted": 15,
  "timeRemaining": 15,
  "claimEligible": false,
  "lastScan": "2025-11-27T19:23:02.989Z",
  "cached": false
}
```

### Get User Status
```
GET /api/user/:walletAddress

Response:
{
  "eligibilityStatus": "Eligible",
  "eligibilityType": ["LP"],
  "lpAmount": 434.683,
  "tokenBalance": 6278.706,
  "daysCompleted": 15,
  "timeRemaining": 15,
  "trackingSince": "2025-11-20T00:00:00.000Z",
  "claimEligible": false,
  "lastScan": "2025-11-27T19:23:02.989Z"
}
```

### Get Statistics
```
GET /api/stats

Response:
{
  "totalUsers": 1250,
  "eligibleForClaim": 342,
  "lpHolders": 580,
  "tokenHolders": 890
}
```

---

## Environment Variables

Create a `.env` file in the root directory:

```bash
# Express Server
PORT=3000
CORS_ORIGINS=http://localhost:5173,http://localhost:3001

# MongoDB
MONGO_URI=mongodb://localhost:27017/zoodi

# Solana RPC Endpoints (comma-separated)
# High Priority: Use multiple endpoints for failover
SOLANA_RPCS=https://mainnet.helius-rpc.com/?api-key=YOUR_KEY,https://api.mainnet-beta.solana.com,https://rpc.ankr.com/solana

# Fallback: Single RPC endpoint (if SOLANA_RPCS not set)
SOLANA_RPC=https://mainnet.helius-rpc.com/?api-key=YOUR_KEY

# Solana Constants
ZOODI_MINT=GrUHrNoaPtEhRZnADCnqPiwYX486L3xfcqG1NiEFCsni
ZOODI_SOL_POOL=3vKja9px5R1PCU4dFtgUkdv8XnwMGG2NHn9rzniHMZDX
RAYDIUM_CLMM_PROGRAM=CAMMCzo5YL8w4VFF8KVHrK22GGUsp5VTaW7grrKgrWqK

# RPC Pool Configuration (optional)
RPC_POOL_ATTEMPTS=4                  # Number of retry attempts per endpoint
RPC_POOL_BASE_DELAY_MS=200           # Base delay (ms) for exponential backoff

# RPC Pagination & Chunking (optional)
RPC_PAGE_SIZE=1000                   # Page size for getProgramAccountsV2
RPC_CHUNK_SIZE=1000                  # Chunk size for getMultipleAccountsInfo

# Cron Configuration (optional)
CRON_BATCH_LIMIT=20                  # Users per batch (default: 20)
CRON_BATCH_DELAY_MS=5000             # Delay between batches in ms (default: 5000)
```

### Example `.env` File
```bash
PORT=3000
CORS_ORIGINS=http://localhost:5173
MONGO_URI=mongodb://localhost:27017/zoodi
SOLANA_RPCS=https://mainnet.helius-rpc.com/?api-key=02d45f55-f...,https://api.mainnet-beta.solana.com
ZOODI_MINT=GrUHrNoaPtEhRZnADCnqPiwYX486L3xfcqG1NiEFCsni
ZOODI_SOL_POOL=3vKja9px5R1PCU4dFtgUkdv8XnwMGG2NHn9rzniHMZDX
RAYDIUM_CLMM_PROGRAM=CAMMCzo5YL8w4VFF8KVHrK22GGUsp5VTaW7grrKgrWqK
RPC_PAGE_SIZE=800
RPC_CHUNK_SIZE=800
CRON_BATCH_LIMIT=50
CRON_BATCH_DELAY_MS=5000
```

---

## Installation & Setup

### Prerequisites
- Node.js 16+ and npm
- MongoDB (local or Atlas)
- Solana RPC endpoint(s) (Helius recommended)

### Steps

1. **Clone & Install**
```bash
cd zoodi-backend
npm install
```

2. **Configure Environment**
```bash
# Copy and edit .env file
cp .env.example .env
# Edit .env with your MongoDB URI, RPC endpoints, and wallet addresses
```

3. **Start Development Server**
```bash
npm run dev
```

4. **Start Production Server**
```bash
npm start
```

The server will listen on `http://localhost:3000` by default.

---

## Running Tests & Diagnostics

### Single Wallet Test
Test eligibility check for one wallet:

```bash
node ./scripts/check_wallet.js <wallet_address>

# Example:
node ./scripts/check_wallet.js HAzWB9uik5fUmA6c9ErcMnSrHDDFuR7L7YTHNf2nAqNu
```

**Output:**
```
✅ Token balance: 6278.706 ZOODI
✅ LP positions found: 3
✅ Total LP Liquidity: 434.683
✅ Total processing time: 1741ms
```

### Cron Batch Test
Simulate the daily cron job for N users:

```bash
node ./scripts/run_daily_check_once.js <count>

# Example: Test with 5 users
node ./scripts/run_daily_check_once.js 5

# Example: Test with 20 users
node ./scripts/run_daily_check_once.js 20
```

**Output:**
```
Daily check batch: 5 users
========================================
🔍 Scanning wallet for LP positions...
Found 955 regular token accounts
Found 170 Token-2022 accounts
Found 135 candidate position tokens...
✅ Total LP positions found: 3
========================================
Daily batch check completed
```

---

## Performance Tuning

### For Large Wallets (1000+ token accounts)

If you see `getMultipleAccountsInfo` or `getProgramAccounts` errors:

```bash
# Reduce chunk/page sizes
RPC_PAGE_SIZE=500
RPC_CHUNK_SIZE=500
RPC_POOL_ATTEMPTS=6
RPC_POOL_BASE_DELAY_MS=300
```

### For Rate-Limiting (many 429 errors)

```bash
# Increase cron batch delays, reduce batch size
CRON_BATCH_LIMIT=10
CRON_BATCH_DELAY_MS=10000
RPC_POOL_ATTEMPTS=8
RPC_POOL_BASE_DELAY_MS=500
```

### For Production / Mainnet

```bash
# Use multiple reliable RPC endpoints
SOLANA_RPCS=https://mainnet.helius-rpc.com/?api-key=KEY1,https://api.mainnet-beta.solana.com,https://rpc.ankr.com/solana

# Conservative batching
CRON_BATCH_LIMIT=50
CRON_BATCH_DELAY_MS=5000
RPC_PAGE_SIZE=1000
RPC_CHUNK_SIZE=1000
```

---

## Project Structure

```
zoodi-backend/
├── src/
│   ├── app.js                           # Express app entry point
│   ├── config/
│   │   ├── db.js                        # MongoDB connection & options
│   │   ├── solana.js                    # Solana program IDs & mints
│   │   └── rpcPool.js                   # Multi-endpoint RPC pool with failover
│   ├── controllers/
│   │   └── eligibilityController.js     # Route handlers
│   ├── models/
│   │   └── User.js                      # User MongoDB schema
│   ├── services/
│   │   ├── eligibilityService.js        # Core eligibility detection logic
│   │   └── cronService.js               # Daily batch cron scheduler
│   └── utils/
│       └── batchUtils.js                # Batch processing utilities
├── scripts/
│   ├── check_wallet.js                  # Single wallet test script
│   └── run_daily_check_once.js          # Cron batch test script
├── .env                                 # Environment variables (not in git)
├── .gitignore                           # Git ignore rules
├── package.json                         # NPM dependencies
└── README.md                            # This file
```

---

## Troubleshooting

### 1. CORS Error
```
❌ CORS error - Backend blocks request from http://localhost:5173
```

**Solution:** Ensure `CORS_ORIGINS` includes your frontend URL:
```bash
CORS_ORIGINS=http://localhost:5173
```

### 2. MongoDB Connection Timeout
```
❌ MongooseError: Operation `users.findOne()` buffering timed out
```

**Solution:** Check MongoDB is running; increase timeouts in `.env`:
```bash
MONGO_URI=mongodb://localhost:27017/zoodi
```

### 3. RPC Rate-Limiting (429 Errors)
```
❌ Server responded with 429 Too Many Requests
```

**Solutions:**
- Add multiple RPC endpoints: `SOLANA_RPCS=url1,url2,url3`
- Increase `RPC_POOL_BASE_DELAY_MS` and `CRON_BATCH_DELAY_MS`
- Reduce `CRON_BATCH_LIMIT` to process fewer users per batch

### 4. Deprioritized Request (Large Wallets)
```
❌ Request deprioritized... Please use getProgramAccountsV2
```

**Solution:** Already handled with paginated fallback; tuning helps:
```bash
RPC_PAGE_SIZE=800
RPC_CHUNK_SIZE=800
```

---

## Key Concepts

### LP Token Detection
- Scans for position NFTs (decimals=0, amount=1) in both Token and Token-2022 programs
- Derives Raydium CLMM position PDAs using seeds: `['position', mint]` and `['personal_position', mint]`
- Fast-path uses `getProgramAccounts` with memcmp filter on pool ID to reduce RPC calls

### Liquidity Calculation
- Parses u128 liquidity value from position account bytes (offsets 81-97)
- Converts to float by dividing by 1e9
- **Note:** This is raw liquidity; token amounts (ZOODI/WSOL) require CLMM math and pool state

### 30-Day Eligibility Tracking
- User starts tracking on first check (`startDate`)
- Increments `daysCompleted` each day (checked daily at midnight)
- After 30 days, `claimEligible` is set to `true`
- If balance drops to 0, user is removed from DB (timer resets on next deposit)

---

## Contributing

1. Create a feature branch
2. Make changes and test locally: `npm run dev`
3. Test with scripts: `node ./scripts/check_wallet.js <addr>`
4. Commit with clear messages
5. Push and create a pull request

---

## License

ISC

---

## Contact & Support

For issues or questions:
1. Check `.env` configuration first
2. Review logs: `npm run dev` shows real-time output
3. Run diagnostic scripts to isolate the problem
4. Check RPC endpoint status if seeing rate-limits

---

## Changelog

### v1.0.0 (Nov 2025)
- ✅ Multi-RPC failover with exponential backoff
- ✅ Paginated `getProgramAccountsV2` fallback for large datasets
- ✅ Chunked `getMultipleAccountsInfo` to prevent throttling
- ✅ Configurable RPC pool, pagination, and cron batch parameters
- ✅ CORS support for frontend integration
- ✅ Daily cron-based batch processing with retry/backoff
- ✅ MongoDB persistence with fast cache + background refresh
- ✅ Support for Token and Token-2022 account programs
