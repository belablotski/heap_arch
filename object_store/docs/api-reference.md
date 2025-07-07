# REST API Reference

## Table of Contents
1. [Authentication](#authentication)
2. [Bucket Operations](#bucket-operations)
3. [Object Operations](#object-operations)
4. [Multipart Upload](#multipart-upload)
5. [Error Responses](#error-responses)
6. [Rate Limiting](#rate-limiting)

## Base URL

```
https://api.objectstore.example.com/v1
```

## S3 Compatibility

For compatibility with existing S3 clients, we also support S3-style URLs as aliases:

```http
# S3-compatible format (aliases to our primary API)
PUT /{bucket-name}/{object-key}              → PUT /buckets/{bucket-name}/objects/{object-key}
GET /{bucket-name}/{object-key}              → GET /buckets/{bucket-name}/objects/{object-key}
DELETE /{bucket-name}/{object-key}           → DELETE /buckets/{bucket-name}/objects/{object-key}
GET /{bucket-name}?list-type=2               → GET /buckets/{bucket-name}/objects
```

**Note:** The primary API uses explicit resource paths (`/buckets/*/objects/*`) for better REST compliance and operational clarity. S3-style URLs are provided for compatibility but may have limitations with advanced features.

## Authentication

### API Key Authentication

Include your API key in the `Authorization` header:

```http
Authorization: Bearer your-api-key-here
```

### Signed URLs

For temporary access, use signed URLs:

```http
GET /buckets/my-bucket/objects/file.txt?signature=abc123&expires=1640995200
```

### IAM Policies

Access control follows AWS S3-compatible policies:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject"],
      "Resource": "arn:aws:s3:::my-bucket/*"
    }
  ]
}
```

## Bucket Operations

### Create Bucket

Creates a new storage bucket.

```http
PUT /buckets/{bucket-name}
```

**Headers:**
- `Authorization: Bearer {token}` (required)
- `Content-Type: application/json`

**Request Body:**
```json
{
  "region": "us-west-1",
  "storage_class": "standard",
  "versioning": {
    "enabled": true
  },
  "lifecycle": {
    "rules": [
      {
        "id": "transition-to-cold",
        "status": "enabled",
        "transitions": [
          {
            "days": 30,
            "storage_class": "cold"
          }
        ]
      }
    ]
  },
  "cors": {
    "rules": [
      {
        "allowed_origins": ["*"],
        "allowed_methods": ["GET", "PUT", "POST"],
        "allowed_headers": ["*"],
        "max_age_seconds": 3600
      }
    ]
  }
}
```

**Response:**
```http
HTTP/1.1 201 Created
Content-Type: application/json

{
  "bucket_name": "my-bucket",
  "region": "us-west-1",
  "created_at": "2025-07-06T10:30:00Z",
  "bucket_id": "bucket-uuid-here"
}
```

### List Buckets

Lists all buckets accessible to the authenticated user.

```http
GET /buckets
```

**Query Parameters:**
- `limit` (optional): Maximum number of buckets to return (default: 100, max: 1000)
- `continuation_token` (optional): Token for pagination

**Response:**
```json
{
  "buckets": [
    {
      "name": "my-bucket-1",
      "region": "us-west-1",
      "created_at": "2025-07-06T10:30:00Z",
      "storage_class": "standard"
    },
    {
      "name": "my-bucket-2",
      "region": "us-east-1",
      "created_at": "2025-07-05T15:20:00Z",
      "storage_class": "cold"
    }
  ],
  "is_truncated": false,
  "next_continuation_token": null
}
```

### Get Bucket Details

Retrieves detailed information about a specific bucket.

```http
GET /buckets/{bucket-name}
```

**Response:**
```json
{
  "bucket_name": "my-bucket",
  "region": "us-west-1",
  "created_at": "2025-07-06T10:30:00Z",
  "storage_class": "standard",
  "versioning": {
    "enabled": true
  },
  "object_count": 1250,
  "total_size": 1073741824,
  "lifecycle": {
    "rules": [...]
  },
  "cors": {
    "rules": [...]
  }
}
```

### Delete Bucket

Deletes an empty bucket.

```http
DELETE /buckets/{bucket-name}
```

**Response:**
```http
HTTP/1.1 204 No Content
```

**Error Cases:**
- `409 Conflict`: Bucket not empty
- `404 Not Found`: Bucket doesn't exist

## Object Operations

### Upload Object

Uploads a new object or replaces an existing object.

```http
PUT /buckets/{bucket-name}/objects/{object-key}
```

**Headers:**
- `Authorization: Bearer {token}` (required)
- `Content-Type: {mime-type}` (optional)
- `Content-Length: {size}` (required)
- `Content-MD5: {md5-hash}` (optional, for integrity)
- `Cache-Control: {directives}` (optional)
- `x-amz-meta-{name}: {value}` (optional, custom metadata)
- `x-amz-storage-class: {class}` (optional: standard, cold, archive)

**Request Body:** Binary object data

**Response:**
```http
HTTP/1.1 200 OK
ETag: "d41d8cd98f00b204e9800998ecf8427e"
Content-Type: application/json

{
  "object_id": "obj-uuid-here",
  "etag": "d41d8cd98f00b204e9800998ecf8427e",
  "size": 1024,
  "version_id": "v1-uuid-here"
}
```

### Download Object

Downloads an object.

```http
GET /buckets/{bucket-name}/objects/{object-key}
```

**Query Parameters:**
- `version_id` (optional): Specific version to retrieve
- `response-content-type` (optional): Override content type
- `response-content-disposition` (optional): Override disposition

**Headers:**
- `Range: bytes={start}-{end}` (optional, for partial downloads)
- `If-Match: {etag}` (optional, conditional request)
- `If-None-Match: {etag}` (optional, conditional request)
- `If-Modified-Since: {date}` (optional, conditional request)
- `If-Unmodified-Since: {date}` (optional, conditional request)

**Response:**
```http
HTTP/1.1 200 OK
Content-Type: application/octet-stream
Content-Length: 1024
ETag: "d41d8cd98f00b204e9800998ecf8427e"
Last-Modified: Fri, 06 Jul 2025 10:30:00 GMT
x-amz-meta-custom: value
x-amz-storage-class: standard
x-amz-version-id: v1-uuid-here

[Binary object data]
```

**Partial Content Response:**
```http
HTTP/1.1 206 Partial Content
Content-Type: application/octet-stream
Content-Length: 512
Content-Range: bytes 0-511/1024
ETag: "d41d8cd98f00b204e9800998ecf8427e"

[Partial binary data]
```

### Get Object Metadata

Retrieves object metadata without downloading the object body.

```http
HEAD /buckets/{bucket-name}/objects/{object-key}
```

**Response:**
```http
HTTP/1.1 200 OK
Content-Type: application/octet-stream
Content-Length: 1024
ETag: "d41d8cd98f00b204e9800998ecf8427e"
Last-Modified: Fri, 06 Jul 2025 10:30:00 GMT
x-amz-meta-custom: value
x-amz-storage-class: standard
x-amz-version-id: v1-uuid-here
```

### List Objects

Lists objects in a bucket.

```http
GET /buckets/{bucket-name}/objects
```

**Query Parameters:**
- `prefix` (optional): Filter objects by prefix
- `delimiter` (optional): Group objects by delimiter
- `limit` (optional): Maximum objects to return (default: 1000)
- `continuation_token` (optional): Token for pagination
- `start_after` (optional): Start listing after this key
- `fetch_owner` (optional): Include owner information (default: false)

**Response:**
```json
{
  "bucket_name": "my-bucket",
  "prefix": "logs/",
  "delimiter": "/",
  "max_keys": 1000,
  "is_truncated": false,
  "contents": [
    {
      "key": "logs/2025/07/06/app.log",
      "size": 2048,
      "etag": "abc123",
      "last_modified": "2025-07-06T10:30:00Z",
      "storage_class": "standard",
      "owner": {
        "id": "user-id",
        "display_name": "John Doe"
      }
    }
  ],
  "common_prefixes": [
    {
      "prefix": "logs/2025/07/05/"
    },
    {
      "prefix": "logs/2025/07/04/"
    }
  ],
  "next_continuation_token": null
}
```

### Delete Object

Deletes an object.

```http
DELETE /buckets/{bucket-name}/objects/{object-key}
```

**Query Parameters:**
- `version_id` (optional): Specific version to delete

**Response:**
```http
HTTP/1.1 204 No Content
x-amz-delete-marker: false
x-amz-version-id: v1-uuid-here
```

### Copy Object

Copies an object within the same bucket or to a different bucket.

```http
PUT /buckets/{dest-bucket}/objects/{dest-key}
```

**Headers:**
- `x-amz-copy-source: /{source-bucket}/{source-key}` (required)
- `x-amz-copy-source-version-id: {version}` (optional)
- `x-amz-metadata-directive: COPY|REPLACE` (optional)
- `x-amz-meta-{name}: {value}` (optional, if REPLACE)

**Response:**
```json
{
  "etag": "new-etag-here",
  "last_modified": "2025-07-06T10:30:00Z",
  "version_id": "new-version-id"
}
```

## Multipart Upload

### Initiate Multipart Upload

Starts a multipart upload session.

```http
POST /buckets/{bucket-name}/objects/{object-key}/uploads
```

**Headers:**
- `Content-Type: {mime-type}` (optional)
- `x-amz-meta-{name}: {value}` (optional)
- `x-amz-storage-class: {class}` (optional)

**Response:**
```json
{
  "upload_id": "upload-uuid-here",
  "bucket_name": "my-bucket",
  "object_key": "large-file.zip"
}
```

### Upload Part

Uploads a part of the multipart upload.

```http
PUT /buckets/{bucket-name}/objects/{object-key}/uploads/{upload-id}/parts/{part-number}
```

**Headers:**
- `Content-Length: {size}` (required, 5MB minimum except last part)
- `Content-MD5: {md5-hash}` (optional)

**Request Body:** Binary part data

**Response:**
```http
HTTP/1.1 200 OK
ETag: "part-etag-here"

{
  "etag": "part-etag-here",
  "part_number": 1,
  "size": 5242880
}
```

### List Parts

Lists uploaded parts for a multipart upload.

```http
GET /buckets/{bucket-name}/objects/{object-key}/uploads/{upload-id}/parts
```

**Query Parameters:**
- `limit` (optional): Maximum parts to return
- `part_number_marker` (optional): Start after this part number

**Response:**
```json
{
  "upload_id": "upload-uuid-here",
  "bucket_name": "my-bucket",
  "object_key": "large-file.zip",
  "parts": [
    {
      "part_number": 1,
      "etag": "part1-etag",
      "size": 5242880,
      "last_modified": "2025-07-06T10:30:00Z"
    },
    {
      "part_number": 2,
      "etag": "part2-etag",
      "size": 5242880,
      "last_modified": "2025-07-06T10:31:00Z"
    }
  ],
  "is_truncated": false,
  "next_part_number_marker": null
}
```

### Complete Multipart Upload

Completes a multipart upload by assembling the parts.

```http
POST /buckets/{bucket-name}/objects/{object-key}/uploads/{upload-id}/complete
```

**Request Body:**
```json
{
  "parts": [
    {
      "part_number": 1,
      "etag": "part1-etag"
    },
    {
      "part_number": 2,
      "etag": "part2-etag"
    }
  ]
}
```

**Response:**
```json
{
  "location": "/buckets/my-bucket/objects/large-file.zip",
  "bucket_name": "my-bucket",
  "object_key": "large-file.zip",
  "etag": "final-object-etag",
  "size": 10485760,
  "version_id": "version-uuid-here"
}
```

### Abort Multipart Upload

Cancels a multipart upload and removes uploaded parts.

```http
DELETE /buckets/{bucket-name}/objects/{object-key}/uploads/{upload-id}
```

**Response:**
```http
HTTP/1.1 204 No Content
```

## Error Responses

### Error Format

All errors follow a consistent format:

```json
{
  "error": {
    "code": "BucketNotFound",
    "message": "The specified bucket does not exist",
    "details": {
      "bucket_name": "non-existent-bucket"
    },
    "request_id": "req-uuid-here",
    "timestamp": "2025-07-06T10:30:00Z"
  }
}
```

### Common Error Codes

#### Client Errors (4xx)

| Code | HTTP Status | Description |
|------|-------------|-------------|
| `InvalidRequest` | 400 | Malformed request |
| `InvalidBucketName` | 400 | Bucket name doesn't meet requirements |
| `InvalidObjectKey` | 400 | Object key contains invalid characters |
| `MissingContentLength` | 400 | Content-Length header required |
| `InvalidContentLength` | 400 | Content-Length header invalid |
| `AuthenticationFailed` | 401 | Invalid credentials |
| `AccessDenied` | 403 | Insufficient permissions |
| `BucketNotFound` | 404 | Bucket doesn't exist |
| `ObjectNotFound` | 404 | Object doesn't exist |
| `MethodNotAllowed` | 405 | HTTP method not supported |
| `BucketAlreadyExists` | 409 | Bucket name already taken |
| `BucketNotEmpty` | 409 | Cannot delete non-empty bucket |
| `PreconditionFailed` | 412 | Conditional request failed |
| `InvalidRange` | 416 | Range header invalid |
| `TooManyRequests` | 429 | Rate limit exceeded |

#### Server Errors (5xx)

| Code | HTTP Status | Description |
|------|-------------|-------------|
| `InternalError` | 500 | Internal server error |
| `NotImplemented` | 501 | Feature not implemented |
| `ServiceUnavailable` | 503 | Service temporarily unavailable |
| `InsufficientStorage` | 507 | Insufficient storage space |

### Multipart Upload Errors

| Code | HTTP Status | Description |
|------|-------------|-------------|
| `UploadNotFound` | 404 | Multipart upload doesn't exist |
| `InvalidPartNumber` | 400 | Part number out of range (1-10000) |
| `InvalidPartSize` | 400 | Part size too small (5MB minimum) |
| `InvalidPart` | 400 | Part ETag doesn't match |
| `IncompleteBody` | 400 | Request body incomplete |

## Rate Limiting

### Limits

| Operation | Limit | Window |
|-----------|-------|--------|
| Object GET | 5000/minute | Per API key |
| Object PUT | 1000/minute | Per API key |
| Object DELETE | 1000/minute | Per API key |
| Bucket operations | 100/minute | Per API key |
| List operations | 500/minute | Per API key |

### Headers

Rate limit information is included in response headers:

```http
X-RateLimit-Limit: 5000
X-RateLimit-Remaining: 4999
X-RateLimit-Reset: 1640995260
X-RateLimit-Window: 60
```

### Rate Limit Exceeded Response

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 60
X-RateLimit-Limit: 5000
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1640995260

{
  "error": {
    "code": "TooManyRequests",
    "message": "Rate limit exceeded. Please retry after 60 seconds.",
    "request_id": "req-uuid-here"
  }
}
```

## SDK Examples

### Python

```python
import objectstore

client = objectstore.Client(
    api_key='your-api-key',
    endpoint='https://api.objectstore.example.com/v1'
)

# Upload object
with open('file.txt', 'rb') as f:
    response = client.put_object(
        bucket='my-bucket',
        key='path/to/file.txt',
        body=f,
        content_type='text/plain',
        metadata={'author': 'john'}
    )

print(f"Object uploaded with ETag: {response['etag']}")

# Download object
response = client.get_object(
    bucket='my-bucket',
    key='path/to/file.txt'
)

with open('downloaded.txt', 'wb') as f:
    f.write(response['body'])
```

### JavaScript

```javascript
const ObjectStore = require('@objectstore/client');

const client = new ObjectStore({
  apiKey: 'your-api-key',
  endpoint: 'https://api.objectstore.example.com/v1'
});

// Upload object
const uploadResponse = await client.putObject({
  bucket: 'my-bucket',
  key: 'path/to/file.txt',
  body: fileBuffer,
  contentType: 'text/plain',
  metadata: { author: 'john' }
});

console.log('Object uploaded with ETag:', uploadResponse.etag);

// Download object
const downloadResponse = await client.getObject({
  bucket: 'my-bucket',
  key: 'path/to/file.txt'
});

const buffer = await downloadResponse.body.buffer();
```

### curl Examples

```bash
# Create bucket
curl -X PUT "https://api.objectstore.example.com/v1/buckets/my-bucket" \
  -H "Authorization: Bearer your-api-key" \
  -H "Content-Type: application/json" \
  -d '{"region": "us-west-1"}'

# Upload object
curl -X PUT "https://api.objectstore.example.com/v1/buckets/my-bucket/objects/file.txt" \
  -H "Authorization: Bearer your-api-key" \
  -H "Content-Type: text/plain" \
  --data-binary @file.txt

# Download object
curl -X GET "https://api.objectstore.example.com/v1/buckets/my-bucket/objects/file.txt" \
  -H "Authorization: Bearer your-api-key" \
  -o downloaded-file.txt

# List objects
curl -X GET "https://api.objectstore.example.com/v1/buckets/my-bucket/objects?prefix=logs/" \
  -H "Authorization: Bearer your-api-key"
```
