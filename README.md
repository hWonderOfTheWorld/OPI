# OPI - Open Protocol Indexer

Open Protocol Indexer, OPI, is the **best-in-slot open-source indexing client** for **meta-protocols** on Bitcoin.
OPI uses a fork of **ord 0.9.0** with minimal changes to maintain compatibility with base layer rules. Also, OPI is built with **modularity** in mind. The main indexer indexes all text/json inscriptions and modules can extend it with different meta-protocols.
All modules in OPI have been built with **reorg protection**.

Currently OPI has modules for **brc-20** and **bitmaps**, we'll add new modules over time. Pull Requests are welcomed for other meta-protocols.

## Main Meta-protocol Indexer

**Meta-protocol indexer** sits in the core of OPI. It indexes **all json/text inscriptions** and their **first 2 transfers**.
Transfer limit can be changed via `INDEX_TX_LIMIT` variable in ord fork. This limit has been added since there are some UTXO's with a lot of inscription content and their movement floods transfers tables. Also, base indexing of most protocols only needs first two transfers. BRC-20 becomes invalid after 2 hops, bitmap and SNS validity is calculated at inscription time, etc.

## BRC-20 Indexer / API

**BRC-20 Indexer** is the first module of OPI. It follows the official protocol rules hosted [here](https://layer1.gitbook.io/layer1-foundation/protocols/brc-20/indexing). BRC-20 Indexer saves all historical balance changes and all brc-20 events.

In addition to indexing all events, it also calculates a block hash and cumulative hash of all events for easier db comparison. Here's the pseudocode for hash calculation:
```python
## Calculation starts at block 767430 which is the first inscription block

EVENT_SEPARATOR = '|'
## max_supply, limit_per_mint, amount decimal count is the same as ticker's decimals
## tickers are lowercase
for event in block_events:
  if event is 'deploy-inscribe':
    block_str += 'deploy-inscribe;<inscr_id>;<deployer_pkscript>;<ticker>;<max_supply>;<decimals>;<limit_per_mint>' + EVENT_SEPARATOR
  if event is 'mint-inscribe':
    block_str += 'mint-inscribe;<inscr_id>;<minter_pkscript>;<ticker>;<amount>' + EVENT_SEPARATOR
  if event is 'transfer-inscribe':
    block_str += 'transfer-inscribe;<inscr_id>;<source_pkscript>;<ticker>;<amount>' + EVENT_SEPARATOR
  if event is 'transfer-transfer':
    ## if sent as fee, sent_pkscript is empty
    block_str += 'transfer-transfer;<inscr_id>;<source_pkscript>;<sent_pkscript>;<ticker>;<amount>' + EVENT_SEPARATOR

block_hash = sha256_hex(block_str)
## for first block last_cumulative_hash is empty
cumulative_hash = sha256_hex(last_cumulative_hash + block_hash)
```

**BRC-20 API** exposes activity on block (block events), balance of a wallet at the start of a given height, current balance of a wallet, block hash and cumulative hash at a given block and hash of all current balances.

## Bitmap Indexer / API

**Bitmap Indexer** is the second module of OPI. It follows the official protocol rules hosted [here](https://gitbook.bitmap.land/ruleset/district-ruleset). Bitmap Indexer saves all bitmap-number inscription-id pairs.

In addition to indexing all pairs, it also calculates a block hash and cumulative hash of all events for easier db comparison. Here's the pseudocode for hash calculation:

```python
## Calculation starts at block 767430 which is the first inscription block

EVENT_SEPARATOR = '|'
for bitmap in new_bitmaps_in_block:
  block_str += 'inscribe;<inscr_id>;<bitmap_number>' + EVENT_SEPARATOR

block_hash = sha256_hex(block_str)
## for first block last_cumulative_hash is empty
cumulative_hash = sha256_hex(last_cumulative_hash + block_hash)
```

**Bitmap API** exposes block hash and cumulative hash at a given block, hash of all bitmaps and inscription_id of a given bitmap.

# Setup

OPI uses PostgreSQL as DB. Before running the indexer, setup a PostgreSQL DB (all modules can write into different databases as well as use a single database). Run init_db.sql for each module on their respective database.

**Build ord:**
```bash
cd ord; cargo build --release;
```

**Install node modules**
```bash
cd helpers/main_index; npm install;
cd ../brc20_api; npm install;
cd ../bitmap_api; npm install;
```
*Optional:*
Remove following from `helpers/main_index/node_modules/bitcoinjs-lib/src/payments/p2tr.js`
```js
if (pubkey && pubkey.length) {
  if (!(0, ecc_lib_1.getEccLib)().isXOnlyPoint(pubkey))
    throw new TypeError('Invalid pubkey for p2tr');
}
```
Otherwise, it cannot decode some addresses such as `512057cd4cfa03f27f7b18c2fe45fe2c2e0f7b5ccb034af4dec098977c28562be7a2`

**Install python libraries**
```bash
pip3 install python-dotenv;
pip3 install psycopg2-binary;
```

**Setup .env files**
Copy `.env_sample` in main_index, brc20_index, brc20_api, bitmap_index and bitmap_api as `.env` and fill necessary information.

# Run

**Main Metaprotocol Indexer**
```bash
cd helpers/main_index; node index.js;
```

**BRC-20 Indexer**
```bash
cd helpers/brc20_index; python3 brc20_index.py;
```

**BRC-20 API**
```bash
cd helpers/brc20_api; node api.js;
```

**Bitmap Indexer**
```bash
cd helpers/bitmap_index; python3 bitmap_index.py;
```

**Bitmap API**
```bash
cd helpers/bitmap_api; node api.js;
```