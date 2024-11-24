```sql
CREATE TABLE solana.accounts
(
    `account_type` String,
    `authorized_voters` Array(Tuple(authorized_voter String, epoch Int64)),
    `authorized_withdrawer` String,
    `block_hash` String,
    `block_slot` Int64,
    `block_timestamp` DateTime64(6),
    `commission` Int64,
    `data` Array(Tuple(raw String, encoding String)),
    `epoch_credits` Array(Tuple(credits String, epoch Int64, previous_credits String)),
    `executable` Bool,
    `is_native` Bool,
    `lamports` Decimal(38, 9),
    `last_timestamp` Array(Tuple(slot Int64, timestamp DateTime64(6))),
    `mint` String,
    `node_pubkey` String,
    `owner` String,
    `prior_voters` Array(Tuple(authorized_pubkey String, epoch_of_last_authorized_switch Int64, target_epoch Int64)),
    `program` String,
    `program_data` String,
    `pubkey` String,
    `rent_epoch` Int64,
    `retrieval_timestamp` DateTime64(6),
    `root_slot` Int64,
    `space` Int64,
    `state` String,
    `token_amount` Decimal(38, 9),
    `token_amount_decimals` Int64,
    `tx_signature` String,
    `votes` Array(Tuple(confirmation_count Int64, slot Int64))
)
ENGINE = SharedReplacingMergeTree('/clickhouse/tables/{uuid}/{shard}', '{replica}')
PRIMARY KEY (block_timestamp, block_slot)
ORDER BY (block_timestamp, block_slot, pubkey, block_hash)
SETTINGS index_granularity = 8192
```

```sql
CREATE TABLE solana.block_txn_counts_by_day
(
    `day` Date,
    `block_count` AggregateFunction(uniqCombined(14), String),
    `txn_count` AggregateFunction(sum, Int64)
)
ENGINE = SharedAggregatingMergeTree('/clickhouse/tables/{uuid}/{shard}', '{replica}')
ORDER BY day
SETTINGS index_granularity = 8192
```

```sql
CREATE MATERIALIZED VIEW solana.block_txn_counts_by_day_mv TO solana.block_txn_counts_by_day
(
    `day` DateTime,
    `block_count` AggregateFunction(uniqCombined(14), String),
    `txn_count` AggregateFunction(sum, Int64)
)
AS SELECT
    dateTrunc('day', block_timestamp) AS day,
    uniqCombinedState(14)(block_hash) AS block_count,
    sumState(transaction_count) AS txn_count
FROM solana.blocks
GROUP BY day
```

```sql
CREATE TABLE solana.blocks
(
    `block_hash` String,
    `block_timestamp` DateTime64(6),
    `height` Int64,
    `leader` String,
    `leader_reward` Decimal(38, 9),
    `previous_block_hash` String,
    `slot` Int64,
    `transaction_count` Int64
)
ENGINE = SharedReplacingMergeTree('/clickhouse/tables/{uuid}/{shard}', '{replica}')
PRIMARY KEY (block_timestamp, slot)
ORDER BY (block_timestamp, slot, block_hash)
SETTINGS index_granularity = 8192
```

```sql
CREATE TABLE solana.transactions
(
    `accounts` Array(Tuple(pubkey String, signer Bool, writable Bool)),
    `balance_changes` Array(Tuple(account String, before Decimal(38, 9), after Decimal(38, 9))),
    `block_hash` String,
    `block_slot` Int64,
    `block_timestamp` DateTime64(6),
    `compute_units_consumed` Decimal(38, 9),
    `err` String,
    `fee` Decimal(38, 9),
    `index` Int64,
    `log_messages` Array(String),
    `post_token_balances` Array(Tuple(account_index Int64, mint String, owner String, amount Decimal(76, 38), decimals Int64)),
    `pre_token_balances` Array(Tuple(account_index Int64, mint String, owner String, amount Decimal(76, 38), decimals Int64)),
    `recent_block_hash` String,
    `signature` String,
    `status` String,
    INDEX idx_minmax_slots block_slot TYPE minmax GRANULARITY 8
)
ENGINE = SharedReplacingMergeTree('/clickhouse/tables/{uuid}/{shard}', '{replica}')
PRIMARY KEY (block_timestamp, block_slot)
ORDER BY (block_timestamp, block_slot, signature)
SETTINGS index_granularity = 8192
```

```sql
CREATE TABLE solana.transactions_per_day
(
    `date` Date,
    `count` Int64
)
ENGINE = SharedSummingMergeTree('/clickhouse/tables/{uuid}/{shard}', '{replica}')
ORDER BY date
SETTINGS index_granularity = 8192
```


```sql
CREATE MATERIALIZED VIEW solana.transactions_per_day_mv TO solana.transactions_per_day
(
    `date` Date,
    `count` UInt64
)
AS SELECT
    CAST(block_timestamp, 'Date') AS date,
    count() AS count
FROM solana.transactions
GROUP BY date
```


```sql
CREATE TABLE solana.top_token_creators_by_day
(
    `day` DateTime,
    `creator_address` String,
    `token_count` AggregateFunction(uniqCombined(14), String)
)
ENGINE = SharedAggregatingMergeTree('/clickhouse/tables/{uuid}/{shard}', '{replica}')
ORDER BY (day, creator_address)
SETTINGS index_granularity = 8192
```


```sql
CREATE MATERIALIZED VIEW solana.top_token_creators_by_day_mv TO solana.top_token_creators_by_day
(
    `day` DateTime,
    `creator_address` String,
    `token_count` AggregateFunction(uniqCombined(14), String)
)
AS SELECT
    toStartOfDay(block_timestamp) AS day,
    (creators[1]).1 AS creator_address,
    uniqCombinedState(14)(mint) AS token_count
FROM solana.tokens
GROUP BY
    day,
    creator_address
```


```sql
--Top token creators. This query limits to the last week which you can remove. The numbers are an estimate and use a materialized view. For accurate numbers, use the tokens table e.g. https://crypto.clickhouse.com?query=LS0gdG9wIHRva2VuIGNyZWF0b3JzClNFTEVDVAogICAgY3JlYXRvcnMgWyAxIF0gLjEgYXMgY3JlYXRvcl9hZGRyZXNzLAogICAgdW5pcShtaW50KSBhcyB0b2tlbl9jb3VudApGUk9NCiAgICBzb2xhbmEudG9rZW5zCldIRVJFCiAgICBibG9ja190aW1lc3RhbXAgPj0gdG9kYXkoKSAtIElOVEVSVkFMIDEgV0VFSwogICAgQU5EIGNyZWF0b3JfYWRkcmVzcyA8PiAnJwpHUk9VUCBCWQogICAgMQpPUkRFUiBCWQogICAgMiBkZXNjCkxJTUlUCiAgICAxMDA

SELECT

    creator_address,

    uniqCombinedMerge(14)(token_count) AS token_count

FROM

    solana.top_token_creators_by_day

WHERE

    (creator_address != '')

    AND (day > (today() - toIntervalWeek(1)))

GROUP BY

    creator_address

ORDER BY

    token_count DESC

LIMIT

    100
```


```sql
--Transactions per day. Uses a materialized view which aggregates per day. For more granular counts use the solana.transactions table e.g. https://crypto.clickhouse.com?query=U0VMRUNUIHRvU3RhcnRPZkhvdXIoYmxvY2tfdGltZXN0YW1wKSBhcyBob3VyLCBjb3VudCgpIGFzIGNvdW50IEZST00gc29sYW5hLnRyYW5zYWN0aW9ucyBXSEVSRSBibG9ja190aW1lc3RhbXA6OkRhdGUgPSAnMjAyNC0wNy0yNCcgR1JPVVAgQlkgaG91ciBPUkRFUiBCWSBob3VyIEFTQw

SELECT

    date,

    sum(count) as count

FROM

    solana.transactions_per_day

GROUP BY

    date
```

```sql
--Block and transaction counts. Block counts are an estimate here - we use a materialized view with the uniq function. A more accurate count could be compute using the solana.blocks table but this will be subject to quotas e.g. https://crypto.clickhouse.com?query=U0VMRUNUCiAgZGF0ZV90cnVuYygnZGF5JywgYmxvY2tfdGltZXN0YW1wKSBhcyBkYXksCiAgdW5pcUV4YWN0KGJsb2NrX2hhc2gpIGJsb2NrX2NvdW50LAogIHN1bSh0cmFuc2FjdGlvbl9jb3VudCkgYXMgdHhuX2NvdW50CkZST00KICBzb2xhbmEuYmxvY2tzCldIRVJFCiAgZGF5ID49IHRvZGF5KCkgLSBJTlRFUlZBTCAxIE1PTlRICkdST1VQIEJZCiAgMQ

SELECT

    day,

    uniqCombinedMerge(14)(block_count) AS block_count,

    sumMerge(txn_count) AS txn_count

FROM

    solana.block_txn_counts_by_day

GROUP BY

    day

ORDER BY

    day DESC
```

