# API Reference for Photo Storage System

## Overview

The Photo Storage API provides RESTful endpoints for managing photos, albums, search, and user accounts. All endpoints use JSON for data exchange and require authentication via JWT tokens.

## Base URL
```
Production: https://api.photos.example.com/v1
Staging: https://staging-api.photos.example.com/v1
```

## Authentication

### JWT Token Authentication
```http
Authorization: Bearer <jwt_token>
```

### API Key Authentication (for service-to-service)
```http
X-API-Key: <api_key>
```

## Error Responses

### Standard Error Format
```json
{
  "error": {
    "code": "INVALID_REQUEST",
    "message": "The request is invalid",
    "details": "Missing required field: filename",
    "timestamp": "2025-07-10T12:00:00Z",
    "request_id": "req_123456789"
  }
}
```

### Error Codes
- `AUTHENTICATION_REQUIRED` (401)
- `AUTHORIZATION_FAILED` (403)
- `NOT_FOUND` (404)
- `INVALID_REQUEST` (400)
- `RATE_LIMIT_EXCEEDED` (429)
- `INTERNAL_ERROR` (500)
- `SERVICE_UNAVAILABLE` (503)

## Rate Limiting

### Rate Limit Headers
```http
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 999
X-RateLimit-Reset: 1625097600
X-RateLimit-Retry-After: 60
```

### Rate Limits by Endpoint
- **Upload**: 100 requests/hour per user
- **Read/Download**: 10,000 requests/hour per user  
- **Search**: 1,000 requests/hour per user
- **Metadata**: 5,000 requests/hour per user

## Photo Management

### Upload Photo

#### Single Photo Upload
```http
POST /photos/upload
Content-Type: multipart/form-data
Authorization: Bearer <jwt_token>
```

**Request Body:**
```
photo: <binary_file>
album_id: <optional_album_id>
tags: ["vacation", "beach"]
caption: "Beautiful sunset at the beach"
```

**Response:**
```json
{
  "photo": {
    "id": "photo_123456789",
    "filename": "sunset.jpg",
    "size": 2048576,
    "mime_type": "image/jpeg",
    "width": 1920,
    "height": 1080,
    "uploaded_at": "2025-07-10T12:00:00Z",
    "processing_status": "pending",
    "thumbnails": [],
    "metadata": {
      "camera": {
        "make": "Apple",
        "model": "iPhone 13 Pro"
      },
      "location": {
        "latitude": 37.7749,
        "longitude": -122.4194,
        "city": "San Francisco",
        "country": "United States"
      },
      "date_taken": "2025-07-10T10:30:00Z"
    }
  }
}
```

#### Bulk Photo Upload
```http
POST /photos/upload/bulk
Content-Type: multipart/form-data
Authorization: Bearer <jwt_token>
```

**Request Body:**
```
photos: [<binary_file_1>, <binary_file_2>, ...]
album_id: <optional_album_id>
```

**Response:**
```json
{
  "upload_id": "upload_123456789",
  "status": "processing",
  "total_photos": 25,
  "processed_photos": 0,
  "failed_photos": 0,
  "estimated_completion": "2025-07-10T12:15:00Z"
}
```

### Get Photo Details

```http
GET /photos/{photo_id}
Authorization: Bearer <jwt_token>
```

**Response:**
```json
{
  "photo": {
    "id": "photo_123456789",
    "filename": "sunset.jpg",
    "size": 2048576,
    "mime_type": "image/jpeg",
    "width": 1920,
    "height": 1080,
    "uploaded_at": "2025-07-10T12:00:00Z",
    "processing_status": "completed",
    "urls": {
      "original": "https://cdn.photos.example.com/original/photo_123456789.jpg",
      "thumbnails": {
        "150": "https://cdn.photos.example.com/thumb/150/photo_123456789.webp",
        "300": "https://cdn.photos.example.com/thumb/300/photo_123456789.webp",
        "600": "https://cdn.photos.example.com/thumb/600/photo_123456789.webp"
      }
    },
    "metadata": {
      "camera": {
        "make": "Apple",
        "model": "iPhone 13 Pro",
        "settings": {
          "iso": 100,
          "aperture": 2.8,
          "shutter_speed": "1/125",
          "focal_length": 26
        }
      },
      "location": {
        "latitude": 37.7749,
        "longitude": -122.4194,
        "city": "San Francisco",
        "country": "United States"
      },
      "date_taken": "2025-07-10T10:30:00Z",
      "faces": [
        {
          "face_id": "face_987654321",
          "person_id": "person_111222333",
          "confidence": 0.95,
          "bbox": {
            "x": 100,
            "y": 200,
            "width": 150,
            "height": 200
          }
        }
      ],
      "objects": [
        {
          "label": "sunset",
          "confidence": 0.92
        },
        {
          "label": "ocean",
          "confidence": 0.87
        }
      ],
      "colors": {
        "dominant_colors": [
          {
            "hex": "#FF6B35",
            "percentage": 0.35
          },
          {
            "hex": "#F7931E",
            "percentage": 0.28
          }
        ]
      }
    },
    "tags": ["vacation", "beach", "sunset"],
    "albums": ["album_456789"],
    "caption": "Beautiful sunset at the beach"
  }
}
```

### Update Photo

```http
PUT /photos/{photo_id}
Content-Type: application/json
Authorization: Bearer <jwt_token>
```

**Request Body:**
```json
{
  "caption": "Updated caption",
  "tags": ["vacation", "beach", "sunset", "california"],
  "album_ids": ["album_456789", "album_987654"]
}
```

### Delete Photo

```http
DELETE /photos/{photo_id}
Authorization: Bearer <jwt_token>
```

**Response:**
```json
{
  "message": "Photo deleted successfully",
  "deleted_at": "2025-07-10T12:00:00Z"
}
```

### List User Photos

```http
GET /photos?page=1&limit=50&sort=date_desc&album_id=album_123
Authorization: Bearer <jwt_token>
```

**Query Parameters:**
- `page`: Page number (default: 1)
- `limit`: Photos per page (max: 100, default: 50)
- `sort`: Sort order (`date_asc`, `date_desc`, `filename_asc`, `filename_desc`)
- `album_id`: Filter by album
- `start_date`: Filter photos after this date (ISO 8601)
- `end_date`: Filter photos before this date (ISO 8601)
- `tags`: Comma-separated list of tags

**Response:**
```json
{
  "photos": [
    {
      "id": "photo_123456789",
      "filename": "sunset.jpg",
      "thumbnail_url": "https://cdn.photos.example.com/thumb/300/photo_123456789.webp",
      "date_taken": "2025-07-10T10:30:00Z",
      "size": 2048576
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 50,
    "total_pages": 25,
    "total_photos": 1247,
    "has_next": true,
    "has_previous": false
  }
}
```

## Album Management

### Create Album

```http
POST /albums
Content-Type: application/json
Authorization: Bearer <jwt_token>
```

**Request Body:**
```json
{
  "name": "Summer Vacation 2025",
  "description": "Photos from our trip to California",
  "cover_photo_id": "photo_123456789",
  "is_public": false
}
```

**Response:**
```json
{
  "album": {
    "id": "album_123456789",
    "name": "Summer Vacation 2025",
    "description": "Photos from our trip to California",
    "cover_photo_id": "photo_123456789",
    "cover_photo_url": "https://cdn.photos.example.com/thumb/300/photo_123456789.webp",
    "is_public": false,
    "photo_count": 0,
    "created_at": "2025-07-10T12:00:00Z",
    "updated_at": "2025-07-10T12:00:00Z"
  }
}
```

### Get Album

```http
GET /albums/{album_id}
Authorization: Bearer <jwt_token>
```

### Update Album

```http
PUT /albums/{album_id}
Content-Type: application/json
Authorization: Bearer <jwt_token>
```

### Delete Album

```http
DELETE /albums/{album_id}
Authorization: Bearer <jwt_token>
```

### Add Photos to Album

```http
POST /albums/{album_id}/photos
Content-Type: application/json
Authorization: Bearer <jwt_token>
```

**Request Body:**
```json
{
  "photo_ids": [
    "photo_123456789",
    "photo_987654321"
  ]
}
```

### Remove Photos from Album

```http
DELETE /albums/{album_id}/photos
Content-Type: application/json
Authorization: Bearer <jwt_token>
```

## Search

### Text Search

```http
GET /search?q=sunset+beach&type=text&page=1&limit=20
Authorization: Bearer <jwt_token>
```

**Query Parameters:**
- `q`: Search query
- `type`: Search type (`text`, `visual`, `face`, `hybrid`)
- `page`: Page number
- `limit`: Results per page (max: 100)
- `filters`: JSON-encoded filters

**Response:**
```json
{
  "results": [
    {
      "photo": {
        "id": "photo_123456789",
        "filename": "sunset.jpg",
        "thumbnail_url": "https://cdn.photos.example.com/thumb/300/photo_123456789.webp",
        "date_taken": "2025-07-10T10:30:00Z"
      },
      "relevance_score": 0.95,
      "match_type": "caption"
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total_results": 156,
    "total_pages": 8
  },
  "query_time_ms": 23,
  "suggestions": [
    "sunset beach california",
    "sunset ocean waves"
  ]
}
```

### Visual Similarity Search

```http
POST /search/visual
Content-Type: multipart/form-data
Authorization: Bearer <jwt_token>
```

**Request Body:**
```
image: <binary_file>
limit: 20
```

**Alternative - Search by existing photo:**
```http
GET /search/visual/{photo_id}?limit=20
Authorization: Bearer <jwt_token>
```

### Face Search

```http
GET /search/faces/{person_id}?limit=50
Authorization: Bearer <jwt_token>
```

### Search Suggestions

```http
GET /search/suggestions?q=sun&limit=10
Authorization: Bearer <jwt_token>
```

**Response:**
```json
{
  "suggestions": [
    "sunset",
    "sunrise",
    "sunny day",
    "sun beach",
    "sunshine"
  ]
}
```

## Face Recognition

### List Detected People

```http
GET /faces/people
Authorization: Bearer <jwt_token>
```

**Response:**
```json
{
  "people": [
    {
      "person_id": "person_123456789",
      "name": "John Doe",
      "photo_count": 45,
      "first_seen": "2024-01-15T10:00:00Z",
      "last_seen": "2025-07-10T10:30:00Z",
      "representative_photo": {
        "photo_id": "photo_111222333",
        "face_id": "face_444555666",
        "thumbnail_url": "https://cdn.photos.example.com/faces/face_444555666.jpg"
      }
    }
  ]
}
```

### Name a Person

```http
PUT /faces/people/{person_id}
Content-Type: application/json
Authorization: Bearer <jwt_token>
```

**Request Body:**
```json
{
  "name": "John Doe"
}
```

### Merge People

```http
POST /faces/people/merge
Content-Type: application/json
Authorization: Bearer <jwt_token>
```

**Request Body:**
```json
{
  "primary_person_id": "person_123456789",
  "merge_person_ids": ["person_987654321", "person_456789123"]
}
```

## Sharing

### Create Share Link

```http
POST /photos/{photo_id}/share
Content-Type: application/json
Authorization: Bearer <jwt_token>
```

**Request Body:**
```json
{
  "type": "public",
  "expires_at": "2025-08-10T12:00:00Z",
  "allow_download": true,
  "password": "optional_password"
}
```

**Response:**
```json
{
  "share": {
    "id": "share_123456789",
    "url": "https://photos.example.com/share/abc123def456",
    "type": "public",
    "expires_at": "2025-08-10T12:00:00Z",
    "allow_download": true,
    "view_count": 0,
    "created_at": "2025-07-10T12:00:00Z"
  }
}
```

### Create Album Share

```http
POST /albums/{album_id}/share
Content-Type: application/json
Authorization: Bearer <jwt_token>
```

## User Management

### Get User Profile

```http
GET /user/profile
Authorization: Bearer <jwt_token>
```

**Response:**
```json
{
  "user": {
    "id": "user_123456789",
    "email": "user@example.com",
    "display_name": "John Doe",
    "avatar_url": "https://cdn.photos.example.com/avatars/user_123456789.jpg",
    "storage_used": 5368709120,
    "storage_limit": 107374182400,
    "photo_count": 1247,
    "album_count": 23,
    "created_at": "2024-01-01T00:00:00Z",
    "settings": {
      "auto_backup": true,
      "face_detection": true,
      "location_tracking": true,
      "public_profile": false
    }
  }
}
```

### Update User Settings

```http
PUT /user/settings
Content-Type: application/json
Authorization: Bearer <jwt_token>
```

**Request Body:**
```json
{
  "auto_backup": false,
  "face_detection": true,
  "location_tracking": false
}
```

## WebSocket API

### Real-time Upload Progress

```javascript
const ws = new WebSocket('wss://api.photos.example.com/v1/ws/upload');

ws.onmessage = function(event) {
  const data = JSON.parse(event.data);
  console.log('Upload progress:', data);
};

// Message format:
{
  "type": "upload_progress",
  "upload_id": "upload_123456789",
  "progress": 0.75,
  "processed_files": 15,
  "total_files": 20,
  "current_file": "vacation_photo_15.jpg",
  "estimated_completion": "2025-07-10T12:05:00Z"
}
```

### Real-time Processing Updates

```javascript
const ws = new WebSocket('wss://api.photos.example.com/v1/ws/processing');

// Message format:
{
  "type": "processing_complete",
  "photo_id": "photo_123456789",
  "status": "completed",
  "thumbnails_ready": true,
  "faces_detected": 2,
  "objects_detected": ["sunset", "ocean", "beach"]
}
```

## Webhooks

### Configure Webhook

```http
POST /webhooks
Content-Type: application/json
Authorization: Bearer <jwt_token>
```

**Request Body:**
```json
{
  "url": "https://your-app.com/webhooks/photos",
  "events": [
    "photo.uploaded",
    "photo.processed",
    "album.created"
  ],
  "secret": "your_webhook_secret"
}
```

### Webhook Event Format

```json
{
  "event": "photo.processed",
  "timestamp": "2025-07-10T12:00:00Z",
  "data": {
    "photo_id": "photo_123456789",
    "user_id": "user_987654321",
    "processing_duration_ms": 2345,
    "faces_detected": 2,
    "objects_detected": ["sunset", "ocean"]
  },
  "signature": "sha256=abc123def456..."
}
```

## SDK Examples

### JavaScript/Node.js

```javascript
const PhotosAPI = require('@photos/api-client');

const client = new PhotosAPI({
  apiKey: 'your_api_key',
  baseURL: 'https://api.photos.example.com/v1'
});

// Upload photo
const photo = await client.photos.upload({
  file: fs.createReadStream('photo.jpg'),
  caption: 'Beautiful sunset',
  tags: ['vacation', 'beach']
});

// Search photos
const results = await client.search.text({
  query: 'sunset beach',
  limit: 20
});
```

### Python

```python
from photos_api import PhotosClient

client = PhotosClient(
    api_key='your_api_key',
    base_url='https://api.photos.example.com/v1'
)

# Upload photo
with open('photo.jpg', 'rb') as f:
    photo = client.photos.upload(
        file=f,
        caption='Beautiful sunset',
        tags=['vacation', 'beach']
    )

# Search photos
results = client.search.text(
    query='sunset beach',
    limit=20
)
```

This API provides comprehensive functionality for managing photos at scale while maintaining high performance and reliability.
