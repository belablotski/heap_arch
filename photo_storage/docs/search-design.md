# Search and Indexing System for Photo Storage

## Overview

The search system enables users to find photos using text queries, visual similarity, facial recognition, and advanced filters. It combines traditional text search with modern AI-powered capabilities to provide intuitive and powerful search experiences.

## Search Architecture

```
User Query → Query Parser → Multi-Search Engine → Result Aggregator → Ranked Results
                ↓               ↓                      ↓
            Text Search    Vector Search         Face Search
          (Elasticsearch)   (Pinecone)         (Face Index)
                ↓               ↓                      ↓
            Metadata       Image Embeddings      Face Embeddings
```

## Search Types

### 1. Text-Based Search
```go
type TextSearchEngine struct {
    esClient ElasticsearchClient
    indexName string
}

type TextQuery struct {
    Query     string            `json:"query"`
    Filters   map[string]string `json:"filters"`
    Sort      []SortField       `json:"sort"`
    From      int               `json:"from"`
    Size      int               `json:"size"`
}

func (tse *TextSearchEngine) Search(ctx context.Context, query *TextQuery) (*SearchResults, error) {
    esQuery := map[string]interface{}{
        "query": map[string]interface{}{
            "bool": map[string]interface{}{
                "must": []map[string]interface{}{
                    {
                        "multi_match": map[string]interface{}{
                            "query":  query.Query,
                            "fields": []string{
                                "caption^3",        // Higher weight for captions
                                "objects.label^2",  // Medium weight for objects
                                "scene.category^2", // Medium weight for scenes
                                "filename",         // Lower weight for filename
                                "tags",            // Lower weight for tags
                            },
                            "type": "best_fields",
                            "fuzziness": "AUTO",
                        },
                    },
                },
                "filter": tse.buildFilters(query.Filters),
            },
        },
        "sort": tse.buildSort(query.Sort),
        "from": query.From,
        "size": query.Size,
        "_source": []string{
            "id", "user_id", "filename", "caption", 
            "objects", "scene", "date_time", "location",
        },
    }
    
    response, err := tse.esClient.Search(
        tse.esClient.Search.WithContext(ctx),
        tse.esClient.Search.WithIndex(tse.indexName),
        tse.esClient.Search.WithBody(strings.NewReader(mustJSON(esQuery))),
    )
    
    if err != nil {
        return nil, fmt.Errorf("elasticsearch query failed: %w", err)
    }
    
    return tse.parseResults(response)
}

func (tse *TextSearchEngine) buildFilters(filters map[string]string) []map[string]interface{} {
    var esFilters []map[string]interface{}
    
    for key, value := range filters {
        switch key {
        case "date_range":
            esFilters = append(esFilters, map[string]interface{}{
                "range": map[string]interface{}{
                    "date_time": map[string]interface{}{
                        "gte": value,
                    },
                },
            })
        case "location":
            esFilters = append(esFilters, map[string]interface{}{
                "geo_distance": map[string]interface{}{
                    "distance": "10km",
                    "location.coordinates": value,
                },
            })
        case "user_id":
            esFilters = append(esFilters, map[string]interface{}{
                "term": map[string]interface{}{
                    "user_id": value,
                },
            })
        }
    }
    
    return esFilters
}
```

### 2. Visual Similarity Search
```python
import numpy as np
import faiss
from typing import List, Tuple
import torch
from torchvision import models, transforms

class VisualSearchEngine:
    def __init__(self, embedding_dim: int = 512):
        self.embedding_dim = embedding_dim
        self.device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
        
        # Load pre-trained image encoder
        self.encoder = self._load_image_encoder()
        
        # Initialize FAISS index for fast similarity search
        self.index = faiss.IndexFlatIP(embedding_dim)  # Inner product for cosine similarity
        self.photo_ids = []
        
        # Image preprocessing
        self.transform = transforms.Compose([
            transforms.Resize((224, 224)),
            transforms.ToTensor(),
            transforms.Normalize(mean=[0.485, 0.456, 0.406], 
                               std=[0.229, 0.224, 0.225])
        ])
    
    def _load_image_encoder(self):
        """Load pre-trained ResNet model for feature extraction"""
        model = models.resnet50(pretrained=True)
        # Remove the final classification layer
        model = torch.nn.Sequential(*list(model.children())[:-1])
        model.to(self.device)
        model.eval()
        return model
    
    def encode_image(self, image_path: str) -> np.ndarray:
        """Convert image to feature vector"""
        from PIL import Image
        
        image = Image.open(image_path).convert('RGB')
        image_tensor = self.transform(image).unsqueeze(0).to(self.device)
        
        with torch.no_grad():
            features = self.encoder(image_tensor)
            features = features.squeeze().cpu().numpy()
            
        # Normalize for cosine similarity
        features = features / np.linalg.norm(features)
        return features
    
    def add_photo(self, photo_id: str, embedding: np.ndarray):
        """Add photo embedding to the search index"""
        embedding = embedding.reshape(1, -1).astype('float32')
        self.index.add(embedding)
        self.photo_ids.append(photo_id)
    
    def search_similar(self, query_embedding: np.ndarray, k: int = 20) -> List[Tuple[str, float]]:
        """Find k most similar photos"""
        query_embedding = query_embedding.reshape(1, -1).astype('float32')
        
        # Search for similar embeddings
        similarities, indices = self.index.search(query_embedding, k)
        
        results = []
        for sim, idx in zip(similarities[0], indices[0]):
            if idx < len(self.photo_ids):  # Valid index
                results.append((self.photo_ids[idx], float(sim)))
        
        return results
    
    def search_by_image(self, image_path: str, k: int = 20) -> List[Tuple[str, float]]:
        """Find similar photos by uploading an image"""
        query_embedding = self.encode_image(image_path)
        return self.search_similar(query_embedding, k)
```

### 3. Face Recognition Search
```python
import face_recognition
import pickle
from sklearn.neighbors import NearestNeighbors
import numpy as np

class FaceSearchEngine:
    def __init__(self):
        self.face_encodings = []
        self.photo_face_map = {}  # Maps face_id to (photo_id, face_location)
        self.face_clusters = {}  # Maps person_id to list of face_ids
        self.knn_model = None
        
    def add_faces(self, photo_id: str, face_data: List[dict]):
        """Add face encodings from a photo"""
        for i, face in enumerate(face_data):
            face_id = f"{photo_id}_face_{i}"
            encoding = np.array(face['encoding'])
            
            self.face_encodings.append(encoding)
            self.photo_face_map[face_id] = {
                'photo_id': photo_id,
                'location': face['location'],
                'confidence': face.get('confidence', 1.0)
            }
            
            # Try to cluster this face with existing faces
            self._cluster_face(face_id, encoding)
    
    def _cluster_face(self, face_id: str, encoding: np.ndarray, threshold: float = 0.6):
        """Cluster faces by person using face encodings"""
        if not self.face_encodings:
            # First face - create new person
            person_id = f"person_1"
            self.face_clusters[person_id] = [face_id]
            return
        
        # Compare with existing faces
        encodings_array = np.array(self.face_encodings[:-1])  # Exclude the just-added one
        
        if len(encodings_array) > 0:
            distances = face_recognition.face_distance(encodings_array, encoding)
            min_distance_idx = np.argmin(distances)
            min_distance = distances[min_distance_idx]
            
            if min_distance < threshold:
                # Find which person this face belongs to
                target_face_id = list(self.photo_face_map.keys())[min_distance_idx]
                
                for person_id, face_ids in self.face_clusters.items():
                    if target_face_id in face_ids:
                        self.face_clusters[person_id].append(face_id)
                        return
        
        # No match found - create new person
        person_id = f"person_{len(self.face_clusters) + 1}"
        self.face_clusters[person_id] = [face_id]
    
    def search_by_face(self, query_encoding: np.ndarray, k: int = 20) -> List[dict]:
        """Find photos containing similar faces"""
        if not self.face_encodings:
            return []
        
        encodings_array = np.array(self.face_encodings)
        distances = face_recognition.face_distance(encodings_array, query_encoding)
        
        # Get indices of k most similar faces
        similar_indices = np.argsort(distances)[:k]
        
        results = []
        seen_photos = set()
        
        for idx in similar_indices:
            face_id = list(self.photo_face_map.keys())[idx]
            face_data = self.photo_face_map[face_id]
            photo_id = face_data['photo_id']
            
            # Avoid duplicate photos in results
            if photo_id not in seen_photos:
                results.append({
                    'photo_id': photo_id,
                    'face_id': face_id,
                    'similarity': 1 - distances[idx],  # Convert distance to similarity
                    'face_location': face_data['location']
                })
                seen_photos.add(photo_id)
        
        return results
    
    def get_person_photos(self, person_id: str) -> List[str]:
        """Get all photos containing a specific person"""
        if person_id not in self.face_clusters:
            return []
        
        photo_ids = set()
        for face_id in self.face_clusters[person_id]:
            photo_ids.add(self.photo_face_map[face_id]['photo_id'])
        
        return list(photo_ids)
```

### 4. Advanced Query Processing
```go
type QueryProcessor struct {
    textSearch   TextSearchEngine
    visualSearch VisualSearchEngine
    faceSearch   FaceSearchEngine
    nlpProcessor NLPProcessor
}

type AdvancedQuery struct {
    Text        string                 `json:"text"`
    Image       string                 `json:"image,omitempty"`       // Base64 encoded image
    Filters     map[string]interface{} `json:"filters"`
    Sort        string                 `json:"sort"`
    Limit       int                    `json:"limit"`
    SearchType  string                 `json:"search_type"` // "text", "visual", "face", "hybrid"
}

func (qp *QueryProcessor) ProcessQuery(ctx context.Context, query *AdvancedQuery) (*SearchResults, error) {
    switch query.SearchType {
    case "text":
        return qp.processTextQuery(ctx, query)
    case "visual":
        return qp.processVisualQuery(ctx, query)
    case "face":
        return qp.processFaceQuery(ctx, query)
    case "hybrid":
        return qp.processHybridQuery(ctx, query)
    default:
        return qp.processIntelligentQuery(ctx, query)
    }
}

func (qp *QueryProcessor) processIntelligentQuery(ctx context.Context, query *AdvancedQuery) (*SearchResults, error) {
    // Use NLP to understand the query intent
    intent, entities := qp.nlpProcessor.AnalyzeQuery(query.Text)
    
    var results []*SearchResult
    
    switch intent {
    case "object_search":
        // "Show me photos with cats"
        textQuery := &TextQuery{
            Query: fmt.Sprintf("objects.label:%s", entities["object"]),
            Filters: query.Filters,
            Size: query.Limit,
        }
        textResults, err := qp.textSearch.Search(ctx, textQuery)
        if err != nil {
            return nil, err
        }
        results = textResults.Results
        
    case "location_search":
        // "Photos from Paris"
        query.Filters["location"] = entities["location"]
        textQuery := &TextQuery{
            Query: entities["location"],
            Filters: query.Filters,
            Size: query.Limit,
        }
        textResults, err := qp.textSearch.Search(ctx, textQuery)
        if err != nil {
            return nil, err
        }
        results = textResults.Results
        
    case "person_search":
        // "Photos of John"
        personID := qp.findPersonID(entities["person"])
        if personID != "" {
            photoIDs := qp.faceSearch.GetPersonPhotos(personID)
            results = qp.convertPhotoIDsToResults(photoIDs)
        }
        
    case "temporal_search":
        // "Photos from last summer"
        dateRange := qp.parseDateExpression(entities["time"])
        query.Filters["date_range"] = dateRange
        textQuery := &TextQuery{
            Query: "*",
            Filters: query.Filters,
            Size: query.Limit,
        }
        textResults, err := qp.textSearch.Search(ctx, textQuery)
        if err != nil {
            return nil, err
        }
        results = textResults.Results
    }
    
    return &SearchResults{
        Results: results,
        Total:   len(results),
        Query:   query.Text,
    }, nil
}
```

## Elasticsearch Index Schema

### Photos Index Mapping
```json
{
  "mappings": {
    "properties": {
      "id": {
        "type": "keyword"
      },
      "user_id": {
        "type": "keyword"
      },
      "filename": {
        "type": "text",
        "analyzer": "standard"
      },
      "caption": {
        "type": "text",
        "analyzer": "english",
        "boost": 3.0
      },
      "objects": {
        "type": "nested",
        "properties": {
          "label": {
            "type": "text",
            "analyzer": "keyword",
            "boost": 2.0
          },
          "confidence": {
            "type": "float"
          },
          "bbox": {
            "type": "object",
            "enabled": false
          }
        }
      },
      "scene": {
        "properties": {
          "category": {
            "type": "keyword",
            "boost": 2.0
          },
          "confidence": {
            "type": "float"
          }
        }
      },
      "faces": {
        "type": "nested",
        "properties": {
          "face_id": {
            "type": "keyword"
          },
          "person_id": {
            "type": "keyword"
          },
          "confidence": {
            "type": "float"
          }
        }
      },
      "colors": {
        "properties": {
          "dominant_colors": {
            "type": "nested",
            "properties": {
              "hex": {
                "type": "keyword"
              },
              "percentage": {
                "type": "float"
              }
            }
          },
          "brightness": {
            "type": "float"
          },
          "contrast": {
            "type": "float"
          }
        }
      },
      "location": {
        "properties": {
          "coordinates": {
            "type": "geo_point"
          },
          "city": {
            "type": "keyword"
          },
          "country": {
            "type": "keyword"
          },
          "address": {
            "type": "text"
          }
        }
      },
      "date_time": {
        "type": "date"
      },
      "uploaded_at": {
        "type": "date"
      },
      "tags": {
        "type": "keyword"
      },
      "albums": {
        "type": "keyword"
      },
      "metadata": {
        "properties": {
          "camera": {
            "properties": {
              "make": {"type": "keyword"},
              "model": {"type": "keyword"}
            }
          },
          "settings": {
            "properties": {
              "iso": {"type": "integer"},
              "aperture": {"type": "float"},
              "shutter_speed": {"type": "keyword"},
              "focal_length": {"type": "float"}
            }
          }
        }
      }
    }
  },
  "settings": {
    "number_of_shards": 5,
    "number_of_replicas": 1,
    "analysis": {
      "analyzer": {
        "photo_analyzer": {
          "type": "custom",
          "tokenizer": "standard",
          "filter": [
            "lowercase",
            "stop",
            "snowball"
          ]
        }
      }
    }
  }
}
```

## Search Performance Optimization

### Caching Strategy
```go
type SearchCache struct {
    redis   RedisClient
    ttl     time.Duration
}

func (sc *SearchCache) CacheResults(query string, results *SearchResults) error {
    key := sc.generateCacheKey(query)
    data, err := json.Marshal(results)
    if err != nil {
        return err
    }
    
    return sc.redis.Set(key, data, sc.ttl)
}

func (sc *SearchCache) GetCachedResults(query string) (*SearchResults, error) {
    key := sc.generateCacheKey(query)
    data, err := sc.redis.Get(key)
    if err != nil {
        return nil, err
    }
    
    var results SearchResults
    err = json.Unmarshal(data, &results)
    return &results, err
}

func (sc *SearchCache) generateCacheKey(query string) string {
    hash := sha256.Sum256([]byte(query))
    return fmt.Sprintf("search:%x", hash)
}
```

### Index Optimization
```go
type IndexOptimizer struct {
    esClient ElasticsearchClient
}

func (io *IndexOptimizer) OptimizeIndex(indexName string) error {
    // Force merge to optimize index segments
    _, err := io.esClient.Indices.Forcemerge(
        io.esClient.Indices.Forcemerge.WithIndex(indexName),
        io.esClient.Indices.Forcemerge.WithMaxNumSegments(1),
    )
    
    return err
}

func (io *IndexOptimizer) UpdateIndexSettings(indexName string) error {
    settings := map[string]interface{}{
        "index": map[string]interface{}{
            "refresh_interval": "30s",  // Increase refresh interval for better indexing performance
            "number_of_replicas": "0",  // Reduce replicas during bulk indexing
        },
    }
    
    _, err := io.esClient.Indices.PutSettings(
        io.esClient.Indices.PutSettings.WithIndex(indexName),
        io.esClient.Indices.PutSettings.WithBody(strings.NewReader(mustJSON(settings))),
    )
    
    return err
}
```

## Natural Language Processing

### Query Understanding
```python
import spacy
from datetime import datetime, timedelta
import re

class QueryNLPProcessor:
    def __init__(self):
        self.nlp = spacy.load("en_core_web_sm")
        self.object_keywords = self._load_object_keywords()
        self.location_patterns = self._compile_location_patterns()
        self.time_patterns = self._compile_time_patterns()
    
    def analyze_query(self, query: str) -> tuple:
        """Analyze query to extract intent and entities"""
        doc = self.nlp(query.lower())
        
        intent = self._classify_intent(doc)
        entities = self._extract_entities(doc)
        
        return intent, entities
    
    def _classify_intent(self, doc) -> str:
        """Classify the search intent"""
        query_text = doc.text
        
        # Object/thing search
        if any(token.text in self.object_keywords for token in doc):
            return "object_search"
        
        # Person search
        if any(ent.label_ == "PERSON" for ent in doc.ents):
            return "person_search"
        
        # Location search
        if any(ent.label_ in ["GPE", "LOC"] for ent in doc.ents):
            return "location_search"
        
        # Time-based search
        if any(ent.label_ in ["DATE", "TIME"] for ent in doc.ents):
            return "temporal_search"
        
        # Visual similarity
        if "similar" in query_text or "like this" in query_text:
            return "visual_search"
        
        return "general_search"
    
    def _extract_entities(self, doc) -> dict:
        """Extract relevant entities from the query"""
        entities = {}
        
        # Extract named entities
        for ent in doc.ents:
            if ent.label_ == "PERSON":
                entities["person"] = ent.text
            elif ent.label_ in ["GPE", "LOC"]:
                entities["location"] = ent.text
            elif ent.label_ in ["DATE", "TIME"]:
                entities["time"] = ent.text
        
        # Extract objects
        for token in doc:
            if token.text in self.object_keywords:
                entities["object"] = token.text
        
        return entities
    
    def parse_date_expression(self, time_expr: str) -> dict:
        """Parse natural language time expressions"""
        now = datetime.now()
        
        if "last week" in time_expr:
            start_date = now - timedelta(weeks=1)
            return {"gte": start_date.isoformat()}
        elif "last month" in time_expr:
            start_date = now - timedelta(days=30)
            return {"gte": start_date.isoformat()}
        elif "last year" in time_expr:
            start_date = now - timedelta(days=365)
            return {"gte": start_date.isoformat()}
        elif "summer" in time_expr and "last" in time_expr:
            # Last summer (June-August of previous year)
            year = now.year - 1
            start_date = datetime(year, 6, 1)
            end_date = datetime(year, 8, 31)
            return {
                "gte": start_date.isoformat(),
                "lte": end_date.isoformat()
            }
        
        return {}
```

## Auto-completion and Suggestions

### Search Suggestions
```go
type SearchSuggester struct {
    esClient    ElasticsearchClient
    trie        *Trie
    popularQueries map[string]int
}

func (ss *SearchSuggester) GetSuggestions(prefix string, limit int) ([]string, error) {
    suggestions := []string{}
    
    // Get suggestions from trie (fast prefix matching)
    trieSuggestions := ss.trie.GetSuggestions(prefix, limit/2)
    suggestions = append(suggestions, trieSuggestions...)
    
    // Get suggestions from Elasticsearch completion suggester
    esSuggestions, err := ss.getElasticsearchSuggestions(prefix, limit/2)
    if err == nil {
        suggestions = append(suggestions, esSuggestions...)
    }
    
    // Sort by popularity and remove duplicates
    suggestions = ss.rankSuggestions(suggestions)
    
    if len(suggestions) > limit {
        suggestions = suggestions[:limit]
    }
    
    return suggestions, nil
}

func (ss *SearchSuggester) getElasticsearchSuggestions(prefix string, limit int) ([]string, error) {
    query := map[string]interface{}{
        "suggest": map[string]interface{}{
            "photo_suggest": map[string]interface{}{
                "prefix": prefix,
                "completion": map[string]interface{}{
                    "field": "suggest",
                    "size":  limit,
                },
            },
        },
    }
    
    // Execute suggestion query
    response, err := ss.esClient.Search(
        ss.esClient.Search.WithBody(strings.NewReader(mustJSON(query))),
    )
    
    if err != nil {
        return nil, err
    }
    
    return ss.parseSuggestionResponse(response)
}
```

This comprehensive search and indexing system provides powerful, fast, and intuitive search capabilities for the photo storage platform, combining traditional text search with modern AI-powered features.
