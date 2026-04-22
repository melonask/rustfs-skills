# Known Issues with the rustfs Skill

This file documents errors found in the skill during real-world testing,
how to reproduce them, and the fixes that were verified.

Last verified: 2026-04-22

---

## Issue 1: Missing `force_path_style` in the Rust SDK Example

### File

`references/sdks.md`

### Problem

The Rust (`aws-sdk-rust`) example initialized the S3 client directly from the
shared `SdkConfig`:

```rust
let rustfs_client = Client::new(&sdk_config);
```

Because the AWS SDK for Rust defaults to **virtual-hosted-style** bucket
addressing (`my-bucket.127.0.0.1:9000`), the example **failed** against RustFS
when running on `localhost` or an IP address (it issues requests to
`http://rust-sdk-demo.127.0.0.1:9000/`, which cannot resolve).

### Real-world reproduction

Minimal failing Rust program:

```rust
use aws_config::{BehaviorVersion, Region};
use aws_credential_types::Credentials;
use aws_sdk_s3::Client;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let credentials = Credentials::new("rustfsadmin", "rustfsadmin", None, None, "rustfs");
    let region = Region::new("us-east-1");

    let sdk_config = aws_config::defaults(BehaviorVersion::latest())
        .region(region)
        .credentials_provider(credentials)
        .endpoint_url("http://127.0.0.1:9000")
        .load()
        .await;

    // BROKEN: uses virtual-hosted style by default
    let broken = Client::new(&sdk_config);
    match broken.create_bucket().bucket("demo-broken").send().await {
        Ok(_) => println!("Unexpected success"),
        Err(e) => println!("Failed as expected: {}", e),
    }
    Ok(())
}
```

### Fix applied

Create the client via the S3-specific `Builder` and call
`.force_path_style(true)` before building:

```rust
    let s3_config = aws_sdk_s3::config::Builder::from(&sdk_config)
        .force_path_style(true)
        .build();
    let rustfs_client = Client::from_conf(s3_config);
```

**Verification**: A fresh Rust project with `aws-config = "1"`,
`aws-credential-types = "1"`, `aws-sdk-s3 = "1"` was compiled and executed
against `rustfs/rustfs:latest` running on `127.0.0.1:9000`. Bucket creation
succeeded only after enabling `force_path_style`.

---

## Issue 2: Docker Compose Section Mentioned Non-existent `--profile observability` Flag

### File

`references/installation.md`

### Problem

The Docker Compose section contained:

> If the user wants observability (Grafana, Prometheus, Jaeger, Otel), RustFS provides a `--profile observability` option.

There is **no** `--profile` CLI flag on the `rustfs` binary (verified by
running `rustfs --help` inside `rustfs/rustfs:latest`). The only top-level
flags are `-h/--help` and `-V/--version`. The `server` subcommand accepts
`--address`, `--console-enable`, `--console-address`, etc., but **not**
`--profile`.

### Real-world reproduction

```bash
docker run --rm rustfs/rustfs:latest server --profile observability /data
```

Output:

```
error: unexpected argument '--profile' found
```

### Fix applied

Removed the sentence that mentioned `--profile observability`.
The correct way to add observability is via the `RUSTFS_OBS_ENDPOINT`
environment variable or separate side-car containers.

---

## Issue 3: Skill Recommended `mc` (MinIO Client) Instead of Native `rc`

### File

`SKILL.md` and `references/sdks.md`

### Problem

The skill stated:

> You can use standard AWS S3 SDKs, or the MinIO client (`mc`) mapped to RustFS

While `mc` is technically S3-compatible, the skill simultaneously declares
MinIO dead/deprecated. Recommending the MinIO-branded client is inconsistent
with the "all-RustFS" messaging.

### Fix applied

Replaced `mc alias set rustfs ...` with `rc alias set rustfs ...` in
`SKILL.md`, pointing users to the actively-maintained official RustFS CLI
(`rc`), which is available via Docker (`rustfs/rc:latest`), Homebrew,
Cargo, and GitHub releases.

---

## Not an issue — Migration Guide already uses `rc` correctly

### File

`references/migration.md`

### Note

During initial review the migration guide was suspected of using legacy `mc`
commands. Closer inspection showed it **already** uses the official `rc` CLI
throughout (e.g. `rc alias set`, `rc replicate add`, `rc version enable`).
No changes were required in this file.
