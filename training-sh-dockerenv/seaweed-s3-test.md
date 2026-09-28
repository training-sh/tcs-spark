# Testing SeaweedFS S3 API with AWS CLI

This walkthrough verifies that the SeaweedFS S3-compatible API is working correctly using the AWS CLI.

The SeaweedFS configuration used in this example is:

```text
S3 API:        http://localhost:9001
Access Key:    team
Secret Key:    team1234
Region:        us-east-1
Test Bucket:   polaris-test
```

> SeaweedFS is running in Docker. Port `9001` on the Linux host is mapped to the SeaweedFS S3 API on container port `8333`.

---

## 1. Verify the SeaweedFS Container

Check that the SeaweedFS container is running:

```bash
docker ps --format 'table {{.Names}}\t{{.Image}}\t{{.Ports}}' | grep seaweed
```

The port mapping should include:

```text
0.0.0.0:9001->8333/tcp
0.0.0.0:9002->23646/tcp
```

The ports are used as follows:

| Host Port | Container Port | Purpose |
|---|---:|---|
| `9001` | `8333` | SeaweedFS S3 API |
| `9002` | `23646` | SeaweedFS Admin UI |

---

## 2. Test the S3 Endpoint

Run:

```bash
curl -i http://localhost:9001/
```

A response similar to the following is expected:

```text
HTTP/1.1 403 Forbidden
Content-Type: application/xml
Server: SeaweedFS
```

The XML response may contain:

```xml
<Error>
    <Code>AccessDenied</Code>
    <Message>Access Denied.</Message>
</Error>
```

This is expected because `curl` is accessing the S3 endpoint without AWS authentication.

The important point is that the SeaweedFS S3 service is reachable.

---

# AWS CLI Test

## 3. Configure AWS Credentials

Set the credentials used by the local SeaweedFS installation:

```bash
export AWS_ACCESS_KEY_ID=team
export AWS_SECRET_ACCESS_KEY=team1234
export AWS_DEFAULT_REGION=us-east-1
```

Verify:

```bash
echo "$AWS_ACCESS_KEY_ID"
echo "$AWS_DEFAULT_REGION"
```

Expected:

```text
team
us-east-1
```

> Avoid printing the secret access key.

---

## 4. List S3 Buckets

Use the SeaweedFS endpoint instead of the normal AWS S3 endpoint:

```bash
aws --endpoint-url http://localhost:9001 s3 ls
```

If no buckets exist yet, the command may return an empty result.

The important point is that it completes without an authentication or connection error.

---

# Create a Test Bucket

## 5. Create `polaris-test`

Create a bucket that can later be used by Apache Polaris:

```bash
aws --endpoint-url http://localhost:9001 \
  s3 mb s3://polaris-test
```

Expected output:

```text
make_bucket: polaris-test
```

---

## 6. Verify the Bucket

List the available buckets:

```bash
aws --endpoint-url http://localhost:9001 s3 ls
```

The output should contain:

```text
polaris-test
```

---

# Test Object Storage

## 7. Create a Local Test File

Create a small text file:

```bash
echo "SeaweedFS S3 test" > /tmp/seaweed-test.txt
```

Verify:

```bash
cat /tmp/seaweed-test.txt
```

Expected:

```text
SeaweedFS S3 test
```

---

## 8. Upload the File to SeaweedFS

Upload the file:

```bash
aws --endpoint-url http://localhost:9001 \
  s3 cp /tmp/seaweed-test.txt s3://polaris-test/test.txt
```

Expected output should indicate that the file was uploaded.

---

## 9. List Objects in the Bucket

```bash
aws --endpoint-url http://localhost:9001 \
  s3 ls s3://polaris-test/
```

The output should contain:

```text
test.txt
```

---

## 10. Read the Object Directly

The object can also be copied from S3 to standard output:

```bash
aws --endpoint-url http://localhost:9001 \
  s3 cp s3://polaris-test/test.txt -
```

Expected:

```text
SeaweedFS S3 test
```

---

# Optional Cleanup

Delete the test object:

```bash
aws --endpoint-url http://localhost:9001 \
  s3 rm s3://polaris-test/test.txt
```

Verify that the bucket is empty:

```bash
aws --endpoint-url http://localhost:9001 \
  s3 ls s3://polaris-test/
```

Do **not** delete the `polaris-test` bucket if it will be used for the Apache Polaris test.

---

# SeaweedFS Endpoint Summary

From the Linux host, use:

```text
http://localhost:9001
```

For example:

```bash
aws --endpoint-url http://localhost:9001 s3 ls
```

From another Docker container connected to the same Docker network, use the SeaweedFS container hostname and its internal S3 port:

```text
http://seaweedfs:8333
```

For example, Apache Polaris running on the same Docker network should communicate with SeaweedFS using:

```text
http://seaweedfs:8333
```

rather than:

```text
http://localhost:9001
```

This distinction is important because `localhost` inside the Polaris container refers to the **Polaris container itself**, not the Linux host.

---

# Final Storage Layout

After this test, the environment is:

```text
Linux Host
│
├── localhost:9001
│       │
│       └── SeaweedFS S3 API
│               │
│               └── s3://polaris-test/
│
└── Docker Network: dataeng-network
        │
        ├── seaweedfs
        │      └── S3 API :8333
        │
        └── Apache Polaris
               │
               └── http://seaweedfs:8333
```

The bucket:

```text
s3://polaris-test/
```

can now be used as the storage location for the Apache Polaris test catalog.
