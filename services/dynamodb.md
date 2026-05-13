# DynamoDB

Managed NoSQL key-value + document database. Single-digit-ms latency at any scale. Serverless billing option (on-demand) means $0 idle.

**What it's good for:**
- App state that needs fast lookups by a known key (user profiles, sessions, game state)
- High-write workloads (logs, events, IoT)
- Anything where you'd say "I need Redis but also durability"
- The default DB for serverless backends

**What it's NOT good for:**
- Ad-hoc analytics queries / aggregations (use Athena, Redshift)
- Heavily relational data with complex joins (use RDS / Aurora)
- Full-text search (use OpenSearch)
- Workloads where you don't know your access patterns upfront — DynamoDB punishes you for query patterns you didn't design for

---

## Concepts

| Term | Meaning |
|---|---|
| **Table** | Top-level container. Lives in one Region. |
| **Item** | A row. Has up to 400 KB of attributes (no fixed schema beyond the primary key). |
| **Attribute** | A field on an item. Strongly typed (S, N, B, BOOL, L, M, NULL, SS, NS, BS). |
| **Primary key** | Either just a **partition key** (simple PK) or **partition key + sort key** (composite PK). |
| **Partition key** | A.k.a. hash key. DynamoDB hashes this to decide which physical partition stores the item. Pick a high-cardinality value (user ID, slug). |
| **Sort key** | A.k.a. range key. Items with the same partition key get sorted by this. Lets you query "all items for this PK in a range." |
| **GSI** | Global Secondary Index — alternate access pattern using a different PK/SK combo. Adds cost. |
| **LSI** | Local Secondary Index — alternate sort key (same partition key). Must be created at table creation; can't add later. |
| **On-demand** | Pay-per-request. $0 idle. Auto-scales instantly. Best for variable/unknown traffic. |
| **Provisioned** | Pre-allocate read/write capacity units. Cheaper at steady high load; throttles you if you under-provision. |

---

## Creating a table — Console

1. DynamoDB → **Tables** → **Create table**
2. **Table name:** descriptive, hyphens (`order-d20-characters`)
3. **Partition key:** the field you'll most often look items up by + its type
4. **Sort key:** leave blank for simple key-value; add for compound queries
5. **Customize settings** (so you see the choices)
6. **Table class:** **DynamoDB Standard** (Standard-IA only for cold archival data >30 days)
7. **Capacity mode:** **On-demand** for personal/unpredictable
8. **Encryption:** **AWS owned key** (free)
9. **Deletion protection:** Off for personal projects (turn on for prod tables holding user data)
10. **Tags:** `project=<your-project>` for billing splits
11. **Create table** — ~30s to ACTIVE

---

## Creating a table — CLI

```bash
aws dynamodb create-table \
  --table-name order-d20-characters \
  --attribute-definitions AttributeName=slug,AttributeType=S \
  --key-schema AttributeName=slug,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --tags Key=project,Value=order-d20

# Wait until ACTIVE
aws dynamodb wait table-exists --table-name order-d20-characters
```

Note: `PAY_PER_REQUEST` = on-demand. The other value is `PROVISIONED`.

---

## DynamoDB's typed JSON (the part that surprises everyone)

Every attribute value is wrapped with its **type marker**:

| Marker | Meaning | Python equiv |
|---|---|---|
| `S` | String | `str` |
| `N` | Number (passed as string!) | `int` / `Decimal` |
| `B` | Binary (base64) | `bytes` |
| `BOOL` | Boolean | `bool` |
| `NULL` | Null | `None` |
| `L` | List | `list` |
| `M` | Map | `dict` |
| `SS` | String Set | `set[str]` |
| `NS` | Number Set | `set[Decimal]` |
| `BS` | Binary Set | `set[bytes]` |

So an item like `{"slug": "Bucee", "level": 7, "active": true}` is stored as:

```json
{
  "slug":   {"S": "Bucee"},
  "level":  {"N": "7"},
  "active": {"BOOL": true}
}
```

**Note:** `Number` is always passed as a string. DynamoDB uses arbitrary-precision math. In Python it comes back as `Decimal`.

---

## Reading/writing from Python (boto3)

There are two interfaces in boto3:

### Low-level client (`boto3.client('dynamodb')`)
Speaks the raw typed JSON. Use `TypeDeserializer` / `TypeSerializer` helpers.

```python
import boto3
from boto3.dynamodb.types import TypeDeserializer

ddb = boto3.client("dynamodb")
deser = TypeDeserializer().deserialize

resp = ddb.scan(TableName="order-d20-characters")
items = [{k: deser(v) for k, v in item.items()} for item in resp["Items"]]
```

### High-level resource (`boto3.resource('dynamodb')`)
Auto-converts to/from Python types. Easier for hand-written code:

```python
import boto3

table = boto3.resource("dynamodb").Table("order-d20-characters")
items = table.scan()["Items"]   # Already plain dicts (numbers as Decimal)
```

**Rule of thumb in Lambda:** use the low-level **client** for performance (slightly faster cold start, no extra abstraction layer). Use the **resource** for scripts and prototyping.

---

## Common operations cheat sheet

```bash
# Get one item
aws dynamodb get-item \
  --table-name mytable \
  --key '{"slug":{"S":"Bucee_Smashcan"}}'

# Put / overwrite
aws dynamodb put-item \
  --table-name mytable \
  --item '{"slug":{"S":"Bucee"},"level":{"N":"8"}}'

# Update one attribute
aws dynamodb update-item \
  --table-name mytable \
  --key '{"slug":{"S":"Bucee"}}' \
  --update-expression "SET #lv = :v" \
  --expression-attribute-names '{"#lv":"level"}' \
  --expression-attribute-values '{":v":{"N":"9"}}'

# Delete one item
aws dynamodb delete-item \
  --table-name mytable \
  --key '{"slug":{"S":"Bucee"}}'

# Scan whole table (slow! reads everything)
aws dynamodb scan --table-name mytable

# Query by partition key + optional sort key range
aws dynamodb query \
  --table-name mytable \
  --key-condition-expression "slug = :s" \
  --expression-attribute-values '{":s":{"S":"Bucee"}}'

# List tables in account
aws dynamodb list-tables

# Describe table (status, item count, sizes)
aws dynamodb describe-table --table-name mytable

# Bulk write (up to 25 items per BatchWriteItem call)
aws dynamodb batch-write-item --request-items file://batch.json

# Delete the whole table
aws dynamodb delete-table --table-name mytable
```

---

## Scan vs. Query (the #1 thing to internalize)

| | **Scan** | **Query** |
|---|---|---|
| Reads | Every item in the table | Only items with the given partition key |
| Cost | Linear in table size | Constant (or proportional to result size) |
| Latency | Grows with data | Always single-digit ms |
| When to use | Admin scripts, small tables, full exports | Almost everything else |

**A real DynamoDB design starts from access patterns, not from data.** Ask "what queries does my app need?" first, then design partition + sort keys to make those queries `Query`-able. Avoid Scan in hot paths.

For order-d20's `GET /api/characters` (return all characters) → Scan is OK because the table is tiny. For `GET /api/sessions/<character>/<session_id>` → Query by partition key.

---

## IAM for DynamoDB access

Lambda needs an execution role with DynamoDB permissions to read/write.

**Personal project quick option:** attach the managed policy `AmazonDynamoDBReadOnlyAccess` (grants read on every table — too broad for prod).

**Production-grade least privilege:** inline policy scoped to specific table ARNs:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": [
      "dynamodb:GetItem",
      "dynamodb:Scan",
      "dynamodb:Query"
    ],
    "Resource": [
      "arn:aws:dynamodb:us-east-1:<account-id>:table/order-d20-characters",
      "arn:aws:dynamodb:us-east-1:<account-id>:table/order-d20-characters/index/*"
    ]
  }]
}
```

For writes add `PutItem`, `UpdateItem`, `DeleteItem`, `BatchWriteItem`.

---

## Pricing

**Free tier (always free, not 12-month):**
- 25 GB storage
- 25 read capacity units (provisioned mode only)
- 25 write capacity units (provisioned mode only)
- 2.5M stream reads
- On-demand: NOT in free tier — but at 10 users, costs literally pennies/month

**On-demand pricing (us-east-1):**
- **Reads:** $0.25 per 1M read request units (1 RRU = 1 strongly consistent read of ≤4 KB)
- **Writes:** $1.25 per 1M write request units (1 WRU = 1 write of ≤1 KB)
- **Storage:** $0.25/GB/month

**Provisioned (if you switch):**
- $0.00065 per RCU-hour, $0.00013 per WCU-hour
- Cheaper for steady 24/7 traffic
- Adds operational complexity (need auto-scaling)

**For order-d20 at 10 users:** $0/month.

**Watch out:**
- **GSIs double your write cost** (every write writes to both the table and the index)
- **Streams + DynamoDB triggers** = extra cost
- **TTL** (auto-delete old items) is free but counts against your delete capacity

---

## Common gotchas

**`json.dumps()` blows up with `TypeError: Object of type Decimal is not JSON serializable`**
DynamoDB returns numbers as `Decimal`. Either:
- `json.dumps(obj, default=str)` — quick fix, casts everything to str
- Write a custom JSON encoder that converts Decimal to int/float

**Scan returns empty / partial results**
Scan returns up to 1 MB per call. Check `LastEvaluatedKey` in the response and paginate. The high-level resource has a paginator built in.

**"Provided list of item keys contains duplicates" on BatchGetItem**
Each key in a batch must be unique. Dedupe before sending.

**ProvisionedThroughputExceededException**
Provisioned mode — you're hitting your RCU/WCU limit. Either bump capacity, enable auto-scaling, or switch to on-demand.

**ResourceNotFoundException despite the table existing**
Either wrong region (DDB tables are regional — check your boto3 client region) or table is still in `CREATING` status.

**Eventually consistent reads return stale data**
Default reads are eventually consistent (cheaper, faster). Pass `ConsistentRead=True` to GetItem/Query for strong consistency (costs 2x).

**Updating a top-level attribute resets nested attributes**
`update-expression "SET data = :v"` replaces the entire `data` map. To update nested keys, use `SET data.subkey = :v`.

**Empty strings used to fail; now they're allowed**
Pre-2020 DynamoDB rejected `""`. Now allowed. Still: avoid them in keys.

**Hot partition: throttles even though provisioned capacity isn't maxed**
One partition is getting too many requests because your partition key has low cardinality (e.g. partition_key = "global"). Spread access by including a higher-cardinality value.

---

## Pattern: pre-serialized blob in a `data` attribute

When porting an existing app whose serialization logic is non-trivial (parses raw API responses → computes derived stats → returns JSON for the frontend), the simplest Lambda design is:

1. **Run the serialization locally** (or in a separate "ingest" Lambda)
2. **Store the serialized JSON as a single `data` string attribute** on the DDB item, alongside a few flat columns for list-view projection
3. **The hot-path Lambda is a pass-through** — `get_item` → `json.loads(item["data"])` → return

### Table shape
```
slug (PK)              "Bucee_Smashcan"
name                   "Bucee Smashcan"          <- list view
race                   "Mapach"                  <- list view
classes                "Fighter 7"               <- list view (flat string!)
side                   "light"                   <- list view
data                   "{...big JSON blob...}"   <- detail view
```

### Why this is good

- **Lambda stays tiny** — no need to package the parsing/serialization code in the Lambda zip
- **Cold starts are fast** — boto3 + DDB read, no big dependencies loaded
- **List view is cheap** — `ProjectionExpression` skips the big `data` attribute
- **400 KB item limit is rarely a problem** — frontend-shape JSON is usually 20-50 KB; raw API blobs might exceed limit and should go to S3 instead

### When to outgrow it

- Output shape changes frequently → re-seeding gets annoying → move serialization into a Lambda triggered by ingest events
- Raw blobs >400 KB → put the blob in S3, store the S3 key in DDB
- Same data needs multiple shapes (mobile vs. web) → serialize on read, or store multiple shapes

### Frontend compatibility gotcha

The new backend can return *valid JSON* in *technically the wrong shape* — and you only find out from the frontend. Example: serializer returns `classes: [{name: "Fighter", level: 7}]`, but the legacy frontend expects `classes: "Fighter 7"` (flat string).

**Mitigation:**
- Compare the new endpoint's response to the legacy server's response field-by-field
- The seeding script is the natural place to do the flattening — write tests that pin the exact response shape
- Phase 8 (CI/CD) should run a smoke test that asserts both the keys *and the types* match

---

## Pattern: composite keys for multi-entity tables

When an app has more than one entity type (characters, sessions, events), the right shape is rarely "one table per entity" *or* "everything in one table." It's usually:

- **One table per top-level entity** (characters)
- **One child table** per child entity that you query by parent (sessions, events)
- **Composite key (PK + SK)** on the child tables so a single `Query` returns every child for a parent

### Concrete example

```
characters    PK: slug                                  → one row per character
sessions      PK: slug,         SK: session_id          → many rows per character
events        PK: session_id,   SK: event_id            → many rows per session
```

Why this works:

- **List all sessions for a character:** `Query` on `sessions` with PK = slug. Single call, one partition.
- **List all events for a session:** `Query` on `events` with PK = session_id. Single call, one partition.
- **Newest sessions first:** SK is a date-prefixed id (`S-2026-05-13-001`), so `ScanIndexForward=false` returns reverse-chronological order with no client-side sort.
- **Event ordering:** SK is a time-sortable id (`<ms_hex>-<random_hex>`), so events come back chronological with no sort.

### Why not "single-table design"?

The full Rick Houlihan / "everything in one table" pattern is powerful for very-high-scale, well-understood access patterns. It's also brutal to read 6 months later. At <100 RPS and <10 access patterns, the multi-table layout above is plenty fast and a *lot* easier for a human to reason about.

Single-table makes sense when:
- You've hit the practical Scan/Query limit on per-table partitions
- You need transactional writes across entities (DDB transactions can span up to 100 items but only within one table without extra plumbing)
- You're moving to provisioned capacity and want to share it

Until then, keep entities in separate tables and use composite keys to model parent → many-children.

### Key Condition gotchas

- `KeyConditionExpression` can use `=`, `<`, `<=`, `>`, `>=`, `BETWEEN`, `begins_with` on the SK — and *only* `=` on the PK. You can't `begins_with` on a partition key.
- `ScanIndexForward` is a boolean on the SK direction: `true` = ascending (default), `false` = descending. There's no equivalent for PK.
- If you need to filter by an attribute that's not part of the key, use `FilterExpression` — but understand it filters *after* the read, so you still pay for the items that didn't match.

---

## Pattern: cascade delete (DDB has no foreign keys)

DDB doesn't know that a session's events live in another table. When you delete a character, nothing automatic happens to their sessions or events — you have to do the cascade yourself.

```python
def delete_character(slug):
    # 1. Find all sessions for this character.
    sessions = sessions_table.query(KeyConditionExpression=Key("slug").eq(slug))
    for s in sessions["Items"]:
        # 2. For each session, delete its events with BatchWriteItem (25-item batches).
        last_key = None
        while True:
            kwargs = {"KeyConditionExpression": Key("session_id").eq(s["session_id"])}
            if last_key:
                kwargs["ExclusiveStartKey"] = last_key
            page = events_table.query(**kwargs)
            if page["Items"]:
                with events_table.batch_writer() as batch:
                    for it in page["Items"]:
                        batch.delete_item(Key={
                            "session_id": it["session_id"],
                            "event_id":   it["event_id"],
                        })
            last_key = page.get("LastEvaluatedKey")
            if not last_key:
                break
        # 3. Delete the session row itself.
        sessions_table.delete_item(Key={"slug": slug, "session_id": s["session_id"]})
    # 4. Finally, delete the character row.
    chars_table.delete_item(Key={"slug": slug})
```

### Why it looks this way

- **`batch_writer()` (from the resource API) handles the 25-item batch limit and retries unprocessed items** so you don't have to. Just `.delete_item(Key=...)` inside the `with` block.
- **Pagination via `LastEvaluatedKey`** — DDB caps a single page at 1 MB. Always loop until `LastEvaluatedKey` is absent.
- **Order matters when readers might race the delete**: events → session row → character row, so a reader can never see "session exists, events gone" or "character gone, sessions still listed."

### When to outgrow this

- If cascade volume is huge (thousands of events per character), this becomes a Lambda timeout risk. Switch to async cleanup: write a "delete marker" record, return 202, let a separate Lambda triggered by a DDB Stream do the actual deletions.
- If you need atomicity (all-or-nothing across rows), use `TransactWriteItems` — but you're capped at 100 items per transaction.

---

## Pattern: Decimal coercion for JSON responses

The boto3 *resource* API (`boto3.resource("dynamodb")`) is much nicer to write than the *client* API for writes — but it returns DDB numbers as Python `Decimal` objects, which `json.dumps` can't serialize.

```python
from decimal import Decimal
import json

def _json_default(o):
    if isinstance(o, Decimal):
        return int(o) if o == o.to_integral_value() else float(o)
    return str(o)

body = json.dumps(payload, default=_json_default)
```

### Why this gotcha bites hard

If you forget the encoder and call `json.dumps(item, default=str)` instead, every number becomes a *string* in the response. The API "works" (200 OK, valid JSON), but the frontend silently breaks: `event.payload.result + 5` becomes `"205"` instead of `25`. The bug surfaces far from the cause.

Watch for it specifically in API responses going to a frontend that does arithmetic on event payloads.

### Alternative: low-level client

`boto3.client("dynamodb")` returns the raw DDB wire format (`{"N": "20"}`), which you then deserialize yourself with `TypeDeserializer().deserialize()`. That returns `Decimal` too, but at least the conversion is in *your* code so the encoder is obviously needed.

The resource API is worth it for the cleaner write code; just always pair it with the encoder above.

---

## When DynamoDB is NOT the right answer

| Need | Better choice |
|---|---|
| Complex JOIN-heavy reporting | Aurora / Athena |
| Full-text search | OpenSearch |
| Time-series at scale | Timestream |
| Graph queries | Neptune |
| Tiny cheap key-value with no durability needed | ElastiCache (Redis/Memcached) |
| You don't know your access patterns | Start with Aurora Serverless v2; move to DDB once patterns stabilize |
