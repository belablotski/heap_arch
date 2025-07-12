# Processing Pipeline for Photo Storage System

## Overview

The processing pipeline handles the transformation, analysis, and indexing of uploaded photos. It's designed to be highly scalable, fault-tolerant, and capable of processing millions of photos daily.

## Architecture

```
Upload → Queue → Workers → ML/CV → Index → Notification
   ↓        ↓        ↓        ↓       ↓         ↓
Storage   Kafka   Kubernetes  GPU    Search   WebSocket
```

## Processing Stages

### Stage 1: Upload Validation
```go
type UploadValidator struct {
    maxFileSize   int64
    allowedTypes  []string
    malwareScanner MalwareScanner
}

func (uv *UploadValidator) ValidateUpload(photo *UploadedPhoto) error {
    // File size validation
    if photo.Size > uv.maxFileSize {
        return errors.New("file too large")
    }
    
    // MIME type validation
    if !uv.isAllowedType(photo.MimeType) {
        return errors.New("unsupported file type")
    }
    
    // Malware scanning
    if uv.malwareScanner.IsInfected(photo.Content) {
        return errors.New("malware detected")
    }
    
    // Image integrity check
    if !uv.isValidImage(photo.Content) {
        return errors.New("corrupted image")
    }
    
    return nil
}
```

### Stage 2: Metadata Extraction
```go
type MetadataExtractor struct {
    exifReader ExifReader
    geoDecoder GeoDecoder
}

type PhotoMetadata struct {
    Width        int                `json:"width"`
    Height       int                `json:"height"`
    ColorSpace   string            `json:"color_space"`
    Camera       CameraInfo        `json:"camera"`
    Location     LocationInfo      `json:"location"`
    DateTime     time.Time         `json:"date_time"`
    Orientation  int               `json:"orientation"`
    Flash        bool              `json:"flash"`
    ISO          int               `json:"iso"`
    Aperture     float64           `json:"aperture"`
    ShutterSpeed string            `json:"shutter_speed"`
    FocalLength  float64           `json:"focal_length"`
}

func (me *MetadataExtractor) ExtractMetadata(photo *Photo) (*PhotoMetadata, error) {
    exifData, err := me.exifReader.ReadExif(photo.Content)
    if err != nil {
        return nil, err
    }
    
    metadata := &PhotoMetadata{
        Width:       exifData.ImageWidth,
        Height:      exifData.ImageLength,
        DateTime:    exifData.DateTime,
        Orientation: exifData.Orientation,
        ISO:         exifData.ISO,
        Aperture:    exifData.FNumber,
    }
    
    // Extract GPS coordinates
    if exifData.HasGPS() {
        location, err := me.geoDecoder.DecodeCoordinates(
            exifData.GPSLatitude, 
            exifData.GPSLongitude,
        )
        if err == nil {
            metadata.Location = location
        }
    }
    
    return metadata, nil
}
```

### Stage 3: Thumbnail Generation
```go
type ThumbnailGenerator struct {
    imageProcessor ImageProcessor
    optimizer     ImageOptimizer
}

type ThumbnailSpec struct {
    Width   int
    Height  int
    Quality int
    Format  string
    Crop    CropType
}

var ThumbnailSpecs = []ThumbnailSpec{
    {150, 150, 70, "webp", CropCenter},
    {300, 300, 75, "webp", CropCenter},
    {600, 600, 80, "webp", CropCenter},
    {1200, 1200, 85, "webp", CropNone},
}

func (tg *ThumbnailGenerator) GenerateThumbnails(photo *Photo) ([]*Thumbnail, error) {
    var thumbnails []*Thumbnail
    
    for _, spec := range ThumbnailSpecs {
        thumb, err := tg.generateThumbnail(photo, spec)
        if err != nil {
            return nil, fmt.Errorf("failed to generate %dx%d thumbnail: %w", 
                spec.Width, spec.Height, err)
        }
        
        // Optimize the thumbnail
        optimized, err := tg.optimizer.Optimize(thumb, spec.Quality)
        if err != nil {
            return nil, err
        }
        
        thumbnails = append(thumbnails, optimized)
    }
    
    return thumbnails, nil
}

func (tg *ThumbnailGenerator) generateThumbnail(photo *Photo, spec ThumbnailSpec) (*Thumbnail, error) {
    // Load original image
    img, err := tg.imageProcessor.Decode(photo.Content)
    if err != nil {
        return nil, err
    }
    
    // Apply orientation correction
    img = tg.imageProcessor.FixOrientation(img, photo.Metadata.Orientation)
    
    // Generate thumbnail based on crop type
    var resized image.Image
    switch spec.Crop {
    case CropCenter:
        resized = tg.imageProcessor.ResizeAndCrop(img, spec.Width, spec.Height)
    case CropNone:
        resized = tg.imageProcessor.ResizePreserveAspect(img, spec.Width, spec.Height)
    }
    
    // Encode to specified format
    encoded, err := tg.imageProcessor.Encode(resized, spec.Format)
    if err != nil {
        return nil, err
    }
    
    return &Thumbnail{
        PhotoID: photo.ID,
        Width:   spec.Width,
        Height:  spec.Height,
        Format:  spec.Format,
        Content: encoded,
        Size:    len(encoded),
    }, nil
}
```

### Stage 4: Computer Vision Processing
```python
import torch
import torchvision.transforms as transforms
from transformers import BlipProcessor, BlipForConditionalGeneration
import face_recognition
import numpy as np

class ComputerVisionProcessor:
    def __init__(self):
        self.device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
        
        # Load pre-trained models
        self.object_detector = self._load_object_detector()
        self.scene_classifier = self._load_scene_classifier()
        self.captioning_model = self._load_captioning_model()
        self.face_encoder = face_recognition.api
        
    def process_image(self, image_path: str) -> dict:
        """Process image and extract all visual features"""
        results = {}
        
        # Load image
        image = self._load_image(image_path)
        
        # Object detection
        results['objects'] = self.detect_objects(image)
        
        # Scene classification
        results['scene'] = self.classify_scene(image)
        
        # Face detection and encoding
        results['faces'] = self.detect_faces(image)
        
        # Image captioning
        results['caption'] = self.generate_caption(image)
        
        # Color analysis
        results['colors'] = self.analyze_colors(image)
        
        # Aesthetic scoring
        results['aesthetic_score'] = self.score_aesthetics(image)
        
        return results
    
    def detect_objects(self, image) -> list:
        """Detect objects in the image"""
        with torch.no_grad():
            predictions = self.object_detector(image)
            
        objects = []
        for pred in predictions:
            if pred['confidence'] > 0.7:
                objects.append({
                    'label': pred['label'],
                    'confidence': float(pred['confidence']),
                    'bbox': pred['bbox'].tolist()
                })
                
        return objects
    
    def classify_scene(self, image) -> dict:
        """Classify the scene/setting of the image"""
        with torch.no_grad():
            features = self.scene_classifier.extract_features(image)
            predictions = self.scene_classifier.classify(features)
            
        return {
            'category': predictions[0]['label'],
            'confidence': float(predictions[0]['confidence']),
            'top_3': [
                {
                    'label': p['label'],
                    'confidence': float(p['confidence'])
                }
                for p in predictions[:3]
            ]
        }
    
    def detect_faces(self, image) -> list:
        """Detect and encode faces in the image"""
        # Convert to RGB if needed
        rgb_image = cv2.cvtColor(np.array(image), cv2.COLOR_BGR2RGB)
        
        # Find face locations
        face_locations = face_recognition.face_locations(rgb_image)
        
        # Generate face encodings
        face_encodings = face_recognition.face_encodings(rgb_image, face_locations)
        
        faces = []
        for i, (encoding, location) in enumerate(zip(face_encodings, face_locations)):
            faces.append({
                'face_id': f"face_{i}",
                'encoding': encoding.tolist(),
                'location': {
                    'top': location[0],
                    'right': location[1], 
                    'bottom': location[2],
                    'left': location[3]
                },
                'confidence': 1.0  # face_recognition doesn't provide confidence
            })
            
        return faces
    
    def generate_caption(self, image) -> str:
        """Generate natural language caption for the image"""
        inputs = self.captioning_processor(image, return_tensors="pt")
        
        with torch.no_grad():
            outputs = self.captioning_model.generate(
                **inputs,
                max_length=50,
                num_beams=5,
                early_stopping=True
            )
            
        caption = self.captioning_processor.decode(outputs[0], skip_special_tokens=True)
        return caption
    
    def analyze_colors(self, image) -> dict:
        """Extract dominant colors from the image"""
        from sklearn.cluster import KMeans
        
        # Convert image to array and reshape
        image_array = np.array(image)
        pixels = image_array.reshape(-1, 3)
        
        # Use k-means to find dominant colors
        kmeans = KMeans(n_clusters=5, random_state=42)
        kmeans.fit(pixels)
        
        colors = []
        for i, color in enumerate(kmeans.cluster_centers_):
            colors.append({
                'rgb': color.astype(int).tolist(),
                'hex': '#{:02x}{:02x}{:02x}'.format(*color.astype(int)),
                'percentage': float(np.sum(kmeans.labels_ == i) / len(kmeans.labels_))
            })
        
        # Sort by percentage
        colors.sort(key=lambda x: x['percentage'], reverse=True)
        
        return {
            'dominant_colors': colors,
            'brightness': float(np.mean(image_array)),
            'contrast': float(np.std(image_array))
        }
```

### Stage 5: Search Indexing
```go
type SearchIndexer struct {
    esClient  ElasticsearchClient
    vectorDB  VectorDatabase
}

type PhotoIndex struct {
    ID          string                 `json:"id"`
    UserID      string                 `json:"user_id"`
    Filename    string                 `json:"filename"`
    Caption     string                 `json:"caption"`
    Objects     []ObjectDetection      `json:"objects"`
    Scene       SceneClassification    `json:"scene"`
    Faces       []FaceDetection        `json:"faces"`
    Colors      ColorAnalysis          `json:"colors"`
    Location    LocationInfo           `json:"location"`
    DateTime    time.Time              `json:"date_time"`
    Tags        []string               `json:"tags"`
    Album       []string               `json:"albums"`
    Metadata    PhotoMetadata          `json:"metadata"`
    Embedding   []float32              `json:"embedding"`
}

func (si *SearchIndexer) IndexPhoto(photo *Photo, features *CVFeatures) error {
    // Create search document
    doc := &PhotoIndex{
        ID:        photo.ID,
        UserID:    photo.UserID,
        Filename:  photo.Filename,
        Caption:   features.Caption,
        Objects:   features.Objects,
        Scene:     features.Scene,
        Faces:     features.Faces,
        Colors:    features.Colors,
        Location:  photo.Metadata.Location,
        DateTime:  photo.Metadata.DateTime,
        Tags:      photo.Tags,
        Album:     photo.Albums,
        Metadata:  photo.Metadata,
        Embedding: features.Embedding,
    }
    
    // Index in Elasticsearch for text search
    err := si.esClient.Index("photos", photo.ID, doc)
    if err != nil {
        return fmt.Errorf("failed to index in Elasticsearch: %w", err)
    }
    
    // Store embedding in vector database for similarity search
    err = si.vectorDB.Store(photo.ID, features.Embedding)
    if err != nil {
        return fmt.Errorf("failed to store embedding: %w", err)
    }
    
    return nil
}
```

## Kafka-Based Message Processing

### Topic Configuration
```yaml
topics:
  photo-uploaded:
    partitions: 32
    replication_factor: 3
    cleanup_policy: delete
    retention_ms: 604800000  # 7 days
    
  photo-processing:
    partitions: 64
    replication_factor: 3
    cleanup_policy: delete
    retention_ms: 259200000  # 3 days
    
  thumbnail-generated:
    partitions: 16
    replication_factor: 3
    cleanup_policy: delete
    retention_ms: 86400000   # 1 day
    
  cv-processing:
    partitions: 32
    replication_factor: 3
    cleanup_policy: delete
    retention_ms: 259200000  # 3 days
    
  search-indexing:
    partitions: 16
    replication_factor: 3
    cleanup_policy: delete
    retention_ms: 86400000   # 1 day
```

### Message Schemas
```go
type PhotoUploadedEvent struct {
    PhotoID     string    `json:"photo_id"`
    UserID      string    `json:"user_id"`
    Filename    string    `json:"filename"`
    Size        int64     `json:"size"`
    MimeType    string    `json:"mime_type"`
    StoragePath string    `json:"storage_path"`
    UploadedAt  time.Time `json:"uploaded_at"`
}

type ProcessingCompletedEvent struct {
    PhotoID       string                 `json:"photo_id"`
    Stage         string                 `json:"stage"`
    Status        string                 `json:"status"`
    Results       map[string]interface{} `json:"results"`
    ProcessedAt   time.Time              `json:"processed_at"`
    Duration      time.Duration          `json:"duration"`
    Error         string                 `json:"error,omitempty"`
}
```

### Worker Implementation
```go
type ProcessingWorker struct {
    consumer         kafka.Consumer
    metadataDB      MetadataDB
    objectStore     ObjectStore
    cvProcessor     CVProcessor
    thumbGenerator  ThumbnailGenerator
    searchIndexer   SearchIndexer
    producer        kafka.Producer
}

func (pw *ProcessingWorker) Start(ctx context.Context) error {
    for {
        select {
        case <-ctx.Done():
            return ctx.Err()
        default:
            msg, err := pw.consumer.ReadMessage(time.Second)
            if err != nil {
                continue
            }
            
            err = pw.processMessage(ctx, msg)
            if err != nil {
                log.Error("Failed to process message", "error", err)
                // Send to dead letter queue
                pw.sendToDeadLetterQueue(msg, err)
            }
        }
    }
}

func (pw *ProcessingWorker) processMessage(ctx context.Context, msg *kafka.Message) error {
    var event PhotoUploadedEvent
    err := json.Unmarshal(msg.Value, &event)
    if err != nil {
        return fmt.Errorf("failed to unmarshal message: %w", err)
    }
    
    // Download photo from object store
    photo, err := pw.objectStore.GetPhoto(event.StoragePath)
    if err != nil {
        return fmt.Errorf("failed to download photo: %w", err)
    }
    
    // Extract metadata
    metadata, err := pw.extractMetadata(photo)
    if err != nil {
        return fmt.Errorf("failed to extract metadata: %w", err)
    }
    
    // Generate thumbnails
    thumbnails, err := pw.thumbGenerator.GenerateThumbnails(photo)
    if err != nil {
        return fmt.Errorf("failed to generate thumbnails: %w", err)
    }
    
    // Upload thumbnails
    err = pw.uploadThumbnails(thumbnails)
    if err != nil {
        return fmt.Errorf("failed to upload thumbnails: %w", err)
    }
    
    // Computer vision processing
    cvFeatures, err := pw.cvProcessor.ProcessImage(photo)
    if err != nil {
        return fmt.Errorf("failed to process with CV: %w", err)
    }
    
    // Update database
    err = pw.updatePhotoMetadata(event.PhotoID, metadata, cvFeatures)
    if err != nil {
        return fmt.Errorf("failed to update metadata: %w", err)
    }
    
    // Index for search
    err = pw.searchIndexer.IndexPhoto(photo, cvFeatures)
    if err != nil {
        return fmt.Errorf("failed to index photo: %w", err)
    }
    
    // Send completion event
    completionEvent := ProcessingCompletedEvent{
        PhotoID:     event.PhotoID,
        Stage:       "complete",
        Status:      "success",
        ProcessedAt: time.Now(),
    }
    
    return pw.sendCompletionEvent(completionEvent)
}
```

## Scaling and Performance

### Horizontal Scaling
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: photo-processor
spec:
  replicas: 50
  selector:
    matchLabels:
      app: photo-processor
  template:
    spec:
      containers:
      - name: processor
        image: photo-processor:latest
        resources:
          requests:
            cpu: 1000m
            memory: 2Gi
            nvidia.com/gpu: 1
          limits:
            cpu: 2000m
            memory: 4Gi
            nvidia.com/gpu: 1
        env:
        - name: KAFKA_BROKERS
          value: "kafka-cluster:9092"
        - name: WORKER_TYPE
          value: "cv-processor"
```

### Performance Optimizations

#### GPU Acceleration
```python
class GPUAcceleratedProcessor:
    def __init__(self):
        self.device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
        
        # Load models on GPU
        self.models = {}
        for model_name in ['object_detection', 'scene_classification', 'face_recognition']:
            model = self.load_model(model_name)
            model.to(self.device)
            model.eval()
            self.models[model_name] = model
    
    def batch_process(self, image_batch: List[torch.Tensor]) -> List[dict]:
        """Process multiple images in a single batch for better GPU utilization"""
        batch_tensor = torch.stack(image_batch).to(self.device)
        
        results = []
        with torch.no_grad():
            # Process entire batch at once
            object_predictions = self.models['object_detection'](batch_tensor)
            scene_predictions = self.models['scene_classification'](batch_tensor)
            
            for i, (obj_pred, scene_pred) in enumerate(zip(object_predictions, scene_predictions)):
                results.append({
                    'objects': self.parse_object_predictions(obj_pred),
                    'scene': self.parse_scene_predictions(scene_pred)
                })
        
        return results
```

#### Async Processing
```go
type AsyncProcessor struct {
    workQueue   chan *ProcessingJob
    workers     int
    workerPool  sync.WaitGroup
}

func (ap *AsyncProcessor) Start() {
    for i := 0; i < ap.workers; i++ {
        ap.workerPool.Add(1)
        go ap.worker()
    }
}

func (ap *AsyncProcessor) worker() {
    defer ap.workerPool.Done()
    
    for job := range ap.workQueue {
        start := time.Now()
        
        err := ap.processJob(job)
        
        duration := time.Since(start)
        ap.recordMetrics(job.Type, duration, err)
        
        if err != nil {
            ap.handleError(job, err)
        }
    }
}
```

### Error Handling and Retry Logic

#### Dead Letter Queue
```go
type DeadLetterHandler struct {
    dlqProducer kafka.Producer
    maxRetries  int
}

func (dlh *DeadLetterHandler) HandleFailedMessage(msg *kafka.Message, err error) {
    retryCount := dlh.getRetryCount(msg)
    
    if retryCount >= dlh.maxRetries {
        // Send to dead letter queue
        dlqMsg := &kafka.Message{
            Topic: "processing-dlq",
            Value: msg.Value,
            Headers: []kafka.Header{
                {Key: "original_topic", Value: []byte(msg.TopicPartition.Topic)},
                {Key: "error", Value: []byte(err.Error())},
                {Key: "failed_at", Value: []byte(time.Now().Format(time.RFC3339))},
            },
        }
        
        dlh.dlqProducer.Produce(dlqMsg, nil)
    } else {
        // Retry with exponential backoff
        delay := time.Duration(math.Pow(2, float64(retryCount))) * time.Second
        time.Sleep(delay)
        
        // Re-queue with incremented retry count
        retryMsg := &kafka.Message{
            Topic: msg.TopicPartition.Topic,
            Value: msg.Value,
            Headers: append(msg.Headers, kafka.Header{
                Key:   "retry_count",
                Value: []byte(strconv.Itoa(retryCount + 1)),
            }),
        }
        
        dlh.dlqProducer.Produce(retryMsg, nil)
    }
}
```

## Monitoring and Observability

### Metrics Collection
```go
var (
    processingDuration = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name: "photo_processing_duration_seconds",
            Help: "Time taken to process photos",
        },
        []string{"stage", "status"},
    )
    
    processingThroughput = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "photos_processed_total",
            Help: "Total number of photos processed",
        },
        []string{"stage", "status"},
    )
    
    queueDepth = prometheus.NewGaugeVec(
        prometheus.GaugeOpts{
            Name: "processing_queue_depth",
            Help: "Number of photos waiting in processing queue",
        },
        []string{"queue_name"},
    )
)
```

### Health Checks
```go
func (pw *ProcessingWorker) HealthCheck() error {
    // Check Kafka connectivity
    if !pw.consumer.IsConnected() {
        return errors.New("kafka consumer disconnected")
    }
    
    // Check database connectivity
    if err := pw.metadataDB.Ping(); err != nil {
        return fmt.Errorf("database connection failed: %w", err)
    }
    
    // Check object storage
    if err := pw.objectStore.HealthCheck(); err != nil {
        return fmt.Errorf("object storage unavailable: %w", err)
    }
    
    // Check GPU availability (if using GPU processing)
    if !pw.cvProcessor.IsGPUAvailable() {
        return errors.New("GPU not available")
    }
    
    return nil
}
```

This processing pipeline provides a robust, scalable solution for handling the complex processing requirements of a photo storage system while maintaining high throughput and reliability.
