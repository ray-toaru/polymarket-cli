# GTD expiration implementation plan

## Problem

The CLI accepts `--order-type GTD`, but CLOB limit orders do not currently expose an explicit expiration timestamp. A GTD order needs a timestamp so the order can expire at the intended time.

## Intended CLI

Single limit order:

```bash
polymarket clob create-order \
  --token 48331043336612883... \
  --side buy \
  --price 0.50 \
  --size 10 \
  --order-type GTD \
  --expiration 1767225600
```

Batch limit orders, using one shared expiration for all orders:

```bash
polymarket clob post-orders \
  --tokens "TOKEN1,TOKEN2" \
  --side buy \
  --prices "0.40,0.60" \
  --sizes "10,10" \
  --order-type GTD \
  --expiration 1767225600
```

## Validation rules

- `--expiration` is required when `--order-type GTD`.
- `--expiration` is rejected for non-GTD limit orders.
- `--expiration` is a Unix timestamp in seconds.
- Polymarket applies a 60-second security threshold, so users should use `now + 60 + desired_lifetime_seconds`.

## Code touch points

- `src/commands/clob.rs`
  - Add `expiration: Option<u64>` to `ClobCommand::CreateOrder`.
  - Add `expiration: Option<u64>` to `ClobCommand::PostOrders`.
  - Parse the Unix timestamp into `chrono::DateTime<Utc>`.
  - Validate GTD/non-GTD combinations before signing.
  - Pass the timestamp to the SDK limit order builder via `.expiration(...)`.
- `README.md`
  - Document GTD usage and the `--expiration` flag.

## Suggested helper

```rust
fn parse_expiration(ts: u64) -> Result<DateTime<Utc>> {
    let ts_i64 = i64::try_from(ts)
        .map_err(|_| anyhow::anyhow!("Invalid expiration: timestamp is too large"))?;

    DateTime::<Utc>::from_timestamp(ts_i64, 0)
        .ok_or_else(|| anyhow::anyhow!("Invalid expiration timestamp: {ts}"))
}

fn validate_expiration(is_gtd: bool, expiration: Option<u64>) -> Result<Option<DateTime<Utc>>> {
    match (is_gtd, expiration) {
        (true, Some(ts)) => Ok(Some(parse_expiration(ts)?)),
        (true, None) => anyhow::bail!(
            "--expiration is required when --order-type GTD. Use Unix seconds; for an effective lifetime of N seconds, use now + 60 + N."
        ),
        (false, Some(_)) => anyhow::bail!("--expiration can only be used with --order-type GTD"),
        (false, None) => Ok(None),
    }
}
```

## Acceptance criteria

```bash
cargo fmt
cargo clippy --all-targets --all-features -- -D warnings
cargo test
```

Manual behavior checks:

```bash
# accepted by CLI parsing and order construction
polymarket clob create-order --token TOKEN --side buy --price 0.50 --size 10 --order-type GTD --expiration 1767225600

# rejected before signing
polymarket clob create-order --token TOKEN --side buy --price 0.50 --size 10 --order-type GTD

# rejected before signing
polymarket clob create-order --token TOKEN --side buy --price 0.50 --size 10 --order-type GTC --expiration 1767225600
```
