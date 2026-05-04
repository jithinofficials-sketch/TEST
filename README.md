**# Shopify Web Pixel Bulk Enablement Feature**

> ****Repository****: helixo-co/bucks-analytics

> ****Branch****: `refactor/cleanup-pixel-only-service`

**## Overview**

This feature enables Shopify Web Pixels across multiple stores automatically using background job processing. It processes stores in batches with built-in rate limiting and retry logic.

**## Key Capabilities**

- ***Batch Processing****: Process up to 500 stores per batch
- ***Rate Limiting****: 2-second delay between stores to avoid API limits
- ***Auto-Chaining****: Automatically queues next batch when current completes
- ***Retry Logic****: Handles failures with exponential backoff (3 attempts)
- ***Progress Tracking****: All results stored in MongoDB
- ***API Triggered****: Start processing via HTTP endpoint or CLI

**## How It Works**

```

API Request → Redis Queue → Worker Process → Shopify API → MongoDB Results

```

1. API receives request and adds job to Redis queue

2. Worker picks up job and queries MongoDB for eligible stores

3. Worker processes each store with 2-second delay

4. Worker calls Shopify API to create web pixel

5. Worker saves results to MongoDB

6. Worker automatically chains next batch if stores remain

**## Prerequisites**

- Node.js 18+
- MongoDB (store data and results)
- Redis (job queue)
- Shopify API access tokens for each store

**## Environment Configuration**

```env

MONGO_URI=mongodb://localhost:27017

REDIS_URL=redis://127.0.0.1:6389

SHOPIFY_API_VERSION=2024-01

PORT=3001

NODE_ENV=development

```

**## API Endpoints**

**### Trigger Pixel Enablement**

- ***POST**** `/api/pixel/trigger-pixel-enablement`
- ***Request Body:****

```json

{

"limit": 500,

"retryType": "required_access"

}

```

- ***Parameters:****
- `limit` (optional): Batch size, default 500
- `retryType` (optional): Set to "required_access" to retry failed shops
- ***Single Shop Mode:****

```

POST /api/pixel/trigger-pixel-enablement?shop=mystore.myshopify.com

```

- ***Response:****

```json

{

"message": "Queued background batch processing with limit 500. BullMQ will handle the shops in chunks.",

"checkStatusIn": "BullMQ / MongoDB"

}

```

**### Health Check**

- ***GET**** `/health`

Returns service status.

**## CLI Commands**

```bash

# Trigger batch processing

npm run trigger:pixel-enablement [limit]

# Retry failed shops

npm run trigger:pixel-retry

# Clear pending jobs

npm run clear:pixel-queue

# Check MongoDB stats

npm run debug:mongodb

```

**## MongoDB Schema**

**### Collection: users**

- ***Required Fields:****
- `myshopify_domain`: Store URL (e.g., "store1.myshopify.com")
- `accessToken`: Shopify API access token
- `is_active`: Must be `true` to be processed
- `first_installed`: Date when app was installed
- ***Auto-Populated Fields:****
- `webPixel_id`: Pixel ID after successful creation
- `acces_checking.status`: SUCCESS, FAILED, or ERROR
- `acces_checking.message`: Error message if failed
- `acces_checking.code`: Error code if applicable
- `acces_checking.date`: Processing timestamp
- `webPixel_last_attempt`: Last processing attempt date
- ***Example Document:****

```json

{

"_id": ObjectId("..."),

"myshopify_domain": "store1.myshopify.com",

"accessToken": "shpat_xxxxx",

"is_active": true,

"first_installed": ISODate("2026-03-17T00:00:00.000Z"),

"webPixel_id": "gid://shopify/WebPixel/123456",

"acces_checking": {

"status": "SUCCESS",

"message": null,

"code": null,

"date": ISODate("2026-05-04T00:00:00.000Z")

},

"webPixel_last_attempt": ISODate("2026-05-04T00:00:00.000Z")

}

```

**## Processing Logic**

**### Shop Selection Criteria**

- ***Standard Processing:****
- `is_active: true`
- `first_installed >= 2026-03-17`
- `webPixel_id` does not exist
- `acces_checking` does not exist
- ***Retry Mode (required_access):****
- Same as above, but includes shops where:

- `acces_checking.message` contains "Required access"

- ***Single Shop Mode:****
- Targets specific shop by `myshopify_domain`

**### Error Handling**

- ***ALREADY_EXISTS Error:****
- Worker queries existing pixels via Shopify API
- Matches pixel by domain in settings
- Updates MongoDB with existing pixel ID
- Marks as SUCCESS
- ***Other Errors:****
- Stored in `acces_checking.message`
- Job retries up to 3 times with exponential backoff
- Failed jobs kept for 30 days (last 1000)

**## Monitoring Progress**

**### MongoDB Queries**

```javascript

// Total processed

db.users.countDocuments({ webPixel_id: { $exists: true } })

// Remaining shops

db.users.countDocuments({

is_active: true,

webPixel_id: { $exists: false }

})

// Failed shops

db.users.countDocuments({

"acces_checking.status": "FAILED"

})

// Shops needing retry

db.users.countDocuments({

"acces_checking.message": { $regex: /Required access/i }

})

```

**### Redis Queue Status**

```bash

redis-cli

> LLEN bull:pixel-enablement:waiting

> LLEN bull:pixel-enablement:active

```

**### Application Logs**

Look for these key messages:

- `✅ Pixel Worker initialized.` - Worker started
- `📊 Total shops matching criteria: X` - Batch size
- `[SUCCESS] store.myshopify.com` - Pixel created
- `[FAILED] store.myshopify.com - Error` - Processing failed
- `🔗 Chaining next batch. Remaining: X` - Auto-chaining
- `✅ All batches completed!` - Processing finished

**## Common Error Messages**

| Error | Meaning | Action |

|-------|---------|--------|

| `ALREADY_EXISTS` | Pixel already created | Auto-recovered by worker |

| `Required access` | Missing API scopes | Retry after granting access |

| `401 Unauthorized` | Invalid access token | Update token in MongoDB |

| `403 Forbidden` | Shop frozen/paused | Check shop status |

**## Architecture**

```

┌─────────────────────────────────────────┐

│         Single Node.js Process          │

│                                         │

│  ┌──────────────┐  ┌──────────────┐   │

│  │ Express API  │  │ Pixel Worker │   │

│  │  (Port 3001) │  │ (Background) │   │

│  └──────────────┘  └──────────────┘   │

└─────────────────────────────────────────┘

│                │

└────────┬───────┘

│

┌─────┴─────┐

│   Redis   │

│ (BullMQ)  │

└───────────┘

│

┌─────┴─────┐

│  MongoDB  │

│  (Stores) │

└───────────┘

```

**## Configuration Options**

**### Queue Settings**

File: `src/utils/queue/pixel-queue.ts`

```typescript

const pixelDefaultJobOptions = {

attempts: 3,                    // Retry count

backoff: {

type: "exponential",

delay: 10000                  // 10s initial delay

},

removeOnComplete: true,         // Clean successful jobs

removeOnFail: {

count: 1000,                  // Keep last 1000 failed

age: 2592000                  // Keep for 30 days

}

};

```

**### Batch Size**

File: `src/controllers/pixelController.ts`

```typescript

export const BATCH_SIZE = 500;  // Shops per batch

```

**### Rate Limiting**

File: `src/utils/worker/pixelWorker.ts`

```typescript

await delay(2000);  // 2 seconds between shops

```

**## Troubleshooting**

**### Worker Not Processing**

1. Check if worker initialized:

```

✅ Pixel Worker initialized.

```

2. Verify Redis connection:

```bash

redis-cli ping

# Should return: PONG

```

3. Check MongoDB connection in logs

**### Jobs Stuck in Queue**

Clear all pending jobs:

```bash

npm run clear:pixel-queue

```

**### High Failure Rate**

1. Check MongoDB for error patterns:

```javascript

db.users.aggregate([

{ $match: { "acces_checking.status": "FAILED" } },

{ $group: { _id: "$acces_checking.message", count: { $sum: 1 } } }

])

```

2. Retry shops with "Required access":

```bash

npm run trigger:pixel-retry

```

**## Integration Methods**

**### Method 1: Standalone Service**

Run as separate microservice:

```javascript

const response = await fetch('http://pixel-service:3001/api/pixel/trigger-pixel-enablement', {

method: 'POST',

headers: { 'Content-Type': 'application/json' },

body: JSON.stringify({ limit: 500 })

});

```

**### Method 2: Code Integration**

Copy these files into your app:

- `src/controllers/pixelController.ts`
- `src/utils/redis.ts`
- `src/utils/queue/pixel-queue.ts`
- `src/utils/worker/pixelWorker.ts`
- `src/scripts/*`

Initialize in your app:

```typescript

import { initPixelWorker } from './utils/worker/pixelWorker';

app.listen(3000);

const pixelWorker = initPixelWorker();

```

**## Performance Considerations**

- ***Processing Speed****: ~1800 stores/hour (2s delay per store)
- ***Concurrency****: 1 batch at a time to avoid rate limits
- ***Memory****: Minimal, processes one shop at a time
- ***Redis****: Lightweight queue storage
- ***MongoDB****: Indexed queries on `is_active`, `webPixel_id`, `myshopify_domain`

**## Security Notes**

- Access tokens stored in MongoDB
- API endpoint has no authentication (add if needed)
- Redis connection should be secured in production
- Shopify API calls use HTTPS
- No sensitive data in logs (tokens not logged)

**## Future Improvements**

**### Real-Time Progress Tracking**

Add a progress bar or dashboard to visualize processing status:

- ***Suggested Implementation:****
- WebSocket endpoint for real-time updates
- Progress API endpoint returning:

- Total shops to process

- Shops completed

- Shops failed

- Current batch progress

- Estimated time remaining

- Simple UI dashboard showing:

- Progress bar with percentage

- Success/failure counts

- Recent processing logs

- Current processing speed (shops/minute)

- ***Example Progress Endpoint:****

```typescript

GET /api/pixel/progress

Response:

{

"total": 5000,

"completed": 2500,

"failed": 50,

"inProgress": 500,

"percentage": 50,

"estimatedTimeRemaining": "2 hours",

"processingSpeed": 25

}

```

This would provide better visibility into long-running batch operations and help identify issues early.

**## Version History**

- ***v1.0.0****
- Initial release
- Batch processing with auto-chaining
- API endpoint for triggering
- Retry logic for failed shops
- MongoDB result tracking
- Recovery logic for ALREADY_EXISTS errors
