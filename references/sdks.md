# RustFS SDK Integration Guide

RustFS is 100% S3 compatible. Users should use the official AWS S3 SDKs for their respective languages.
**CRITICAL RULE:** RustFS requires `Path Style` addressing. Virtual Host Style routing will fail unless specifically configured with custom DNS. You MUST configure the S3 client to enforce Path Style.

## Rust (`aws-sdk-rust`)

RustFS is fully compatible with the official AWS SDK for Rust. Configure your `aws_config` with static credentials and the explicit RustFS endpoint URL.

```rust
use aws_config::{BehaviorVersion, Region};
use aws_credential_types::Credentials;
use aws_sdk_s3::Client;
use aws_sdk_s3::primitives::ByteStream;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // 1. Configure credentials (defaults are rustfsadmin/rustfsadmin)
    let credentials = Credentials::new(
        "rustfsadmin",
        "rustfsadmin",
        None,
        None,
        "rustfs"
    );

    // 2. Set Region (Ignored by RustFS but required by the AWS SDK)
    let region = Region::new("us-east-1");

    // 3. Explicitly define the RustFS endpoint
    let endpoint_url = "http://127.0.0.1:9000";

    // 4. Load the configuration
    let sdk_config = aws_config::defaults(BehaviorVersion::latest())
        .region(region)
        .credentials_provider(credentials)
        .endpoint_url(endpoint_url)
        .load()
        .await;

    // 5. Initialize the client with PATH STYLE addressing enforced
    let s3_config = aws_sdk_s3::config::Builder::from(&sdk_config)
        .force_path_style(true)
        .build();
    let rustfs_client = Client::from_conf(s3_config);

    // --- Example: Create a Bucket ---
    rustfs_client
        .create_bucket()
        .bucket("rust-sdk-demo")
        .send()
        .await?;
    println!("Bucket created successfully");

    // --- Example: Upload a File ---
    let data = tokio::fs::read("local.txt").await?;

    rustfs_client
        .put_object()
        .bucket("rust-sdk-demo")
        .key("remote.txt")
        .body(ByteStream::from(data))
        .send()
        .await?;
    println!("Object uploaded successfully");

    Ok(())
}
```

## Python (boto3)

```python
import boto3
from botocore.client import Config

s3 = boto3.client(
    's3',
    endpoint_url='http://127.0.0.1:9000',
    aws_access_key_id='rustfsadmin',
    aws_secret_access_key='rustfsadmin',
    config=Config(signature_version='s3v4'),
    region_name='us-east-1' # Required by boto3, ignored by RustFS
)

# Upload
s3.upload_file('local.txt', 'my-bucket', 'remote.txt')
# Generate Presigned URL
url = s3.generate_presigned_url(ClientMethod='get_object', Params={'Bucket': 'my-bucket', 'Key': 'remote.txt'}, ExpiresIn=600)
```

## JavaScript / TypeScript (Node.js)

Requires `@aws-sdk/client-s3`.

```typescript
import { S3Client, PutObjectCommand } from "@aws-sdk/client-s3";
import { readFileSync } from "fs";

const s3 = new S3Client({
  region: "us-east-1",
  credentials: {
    accessKeyId: "rustfsadmin",
    secretAccessKey: "rustfsadmin",
  },
  endpoint: "http://127.0.0.1:9000",
  forcePathStyle: true, // CRITICAL FOR RUSTFS
});

await s3.send(
  new PutObjectCommand({
    Bucket: "my-bucket",
    Key: "hello.txt",
    Body: readFileSync("hello.txt"),
  }),
);
```

## Golang (aws-sdk-go-v2)

```go
import (
    "context"
    "log"
    "strings"
    "github.com/aws/aws-sdk-go-v2/aws"
    "github.com/aws/aws-sdk-go-v2/config"
    "github.com/aws/aws-sdk-go-v2/credentials"
    "github.com/aws/aws-sdk-go-v2/service/s3"
)

func main() {
    cfg, err := config.LoadDefaultConfig(context.TODO(),
        config.WithRegion("us-east-1"),
        config.WithCredentialsProvider(credentials.NewStaticCredentialsProvider("rustfsadmin", "rustfsadmin", "")),
        config.WithEndpointResolverWithOptions(aws.EndpointResolverWithOptionsFunc(
            func(service, region string, options ...interface{}) (aws.Endpoint, error) {
                return aws.Endpoint{URL: "http://127.0.0.1:9000"}, nil
            })),
    )

    client := s3.NewFromConfig(cfg, func(o *s3.Options) {
        o.UsePathStyle = true // CRITICAL FOR RUSTFS
    })

    _, err = client.CreateBucket(context.TODO(), &s3.CreateBucketInput{
        Bucket: aws.String("go-sdk-rustfs"),
    })
}
```

## Java (AWS SDK v2)

```java
import software.amazon.awssdk.auth.credentials.AwsBasicCredentials;
import software.amazon.awssdk.auth.credentials.StaticCredentialsProvider;
import software.amazon.awssdk.regions.Region;
import software.amazon.awssdk.services.s3.S3Client;
import software.amazon.awssdk.services.s3.model.CreateBucketRequest;
import java.net.URI;

S3Client s3 = S3Client.builder()
    .endpointOverride(URI.create("http://127.0.0.1:9000"))
    .region(Region.US_EAST_1)
    .credentialsProvider(StaticCredentialsProvider.create(AwsBasicCredentials.create("rustfsadmin", "rustfsadmin")))
    .forcePathStyle(true) // CRITICAL FOR RUSTFS
    .build();

s3.createBucket(CreateBucketRequest.builder().bucket("my-bucket").build());
```
