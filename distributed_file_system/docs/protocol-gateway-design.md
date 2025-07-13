# Protocol Gateway Design - NFS and SMB Support

## Overview

Our distributed file system can expose NFS v3/v4 and SMB 2.1/3.x protocol interfaces alongside REST/gRPC APIs. This enables seamless integration with existing enterprise infrastructure while maintaining our storage optimization benefits.

## Architecture Integration

```
┌─────────────────────────────────────────────────────────────────┐
│                    Client Applications                          │
├─────────────────┬─────────────────┬─────────────────────────────┤
│   NFS Clients   │   SMB Clients   │     API Clients             │
│   (mount -t nfs)│   (net use)     │     (REST/gRPC)             │
└─────────────────┴─────────────────┴─────────────────────────────┘
                                │
┌─────────────────────────────────────────────────────────────────┐
│                   Protocol Gateway Layer                        │
├─────────────────┬─────────────────┬─────────────────────────────┤
│   NFS Server    │   SMB Server    │     API Gateway             │
│   (NFSv3/v4)    │   (SMB2/3)      │     (REST/gRPC)             │
└─────────────────┴─────────────────┴─────────────────────────────┘
                                │
┌─────────────────────────────────────────────────────────────────┐
│                    Unified Backend                              │
├─────────────────┬─────────────────┬─────────────────────────────┤
│ Metadata Service│ Dedup Engine    │   Storage Layer             │
│ Session Manager │ Compression     │   (Hot/Warm/Cold)           │
└─────────────────┴─────────────────┴─────────────────────────────┘
```

## Protocol Gateway Implementation

### 1. NFS Server Gateway

#### Core Components
```go
type NFSGateway struct {
    server       *nfs.Server
    fileSystem   FileSystemInterface
    authManager  AuthenticationManager
    sessionMgr   *SessionManager
    cache        *ProtocolCache
    metrics      *NFSMetrics
}

type NFSFileSystem struct {
    dfsClient    *DFSClient
    pathMapper   *PathMapper
    lockManager  *LockManager
    attrCache    *AttributeCache
}

// Implement NFS v4 operations
func (nfs *NFSFileSystem) Open(ctx context.Context, req *nfsv4.OpenRequest) (*nfsv4.OpenResponse, error) {
    // OpenTelemetry tracing
    ctx, span := otel.Tracer("nfs-gateway").Start(ctx, "nfs.open",
        trace.WithAttributes(
            attribute.String("nfs.operation", "open"),
            attribute.String("file.path", req.Path),
            attribute.String("access.mode", req.AccessMode),
        ),
    )
    defer span.End()
    
    // Convert NFS path to DFS path
    dfsPath := nfs.pathMapper.NFSToDFS(req.Path)
    
    // Check permissions using our RBAC system
    user := nfs.authManager.GetUserFromContext(ctx)
    if !nfs.authManager.HasPermission(user, dfsPath, "read") {
        span.RecordError(errors.New("permission denied"))
        return nil, nfs.toNFSError(ErrPermissionDenied)
    }
    
    // Get file metadata from DFS
    fileInfo, err := nfs.dfsClient.GetFileInfo(ctx, &dfs.GetFileInfoRequest{
        Path: dfsPath,
    })
    if err != nil {
        span.RecordError(err)
        return nil, nfs.toNFSError(err)
    }
    
    // Create file handle
    handle := nfs.sessionMgr.CreateHandle(dfsPath, req.AccessMode)
    
    // Record metrics
    nfs.metrics.RecordOperation("open", time.Since(start))
    
    span.SetAttributes(
        attribute.String("file.id", fileInfo.Id),
        attribute.Int64("file.size", fileInfo.Size),
        attribute.String("handle.id", handle.ID),
    )
    
    return &nfsv4.OpenResponse{
        Handle:     handle.ToNFSHandle(),
        Attributes: nfs.toNFSAttributes(fileInfo),
    }, nil
}

func (nfs *NFSFileSystem) Read(ctx context.Context, req *nfsv4.ReadRequest) (*nfsv4.ReadResponse, error) {
    ctx, span := otel.Tracer("nfs-gateway").Start(ctx, "nfs.read",
        trace.WithAttributes(
            attribute.String("handle.id", req.Handle.String()),
            attribute.Int64("read.offset", req.Offset),
            attribute.Int64("read.length", req.Length),
        ),
    )
    defer span.End()
    
    // Get file handle
    handle := nfs.sessionMgr.GetHandle(req.Handle)
    if handle == nil {
        return nil, nfs.toNFSError(ErrBadHandle)
    }
    
    // Read from DFS with range support
    downloadReq := &dfs.DownloadRequest{
        Path: handle.Path,
        Range: &dfs.Range{
            Start: uint64(req.Offset),
            End:   uint64(req.Offset + req.Length),
        },
    }
    
    stream, err := nfs.dfsClient.DownloadFile(ctx, downloadReq)
    if err != nil {
        span.RecordError(err)
        return nil, nfs.toNFSError(err)
    }
    
    var data []byte
    for {
        resp, err := stream.Recv()
        if err == io.EOF {
            break
        }
        if err != nil {
            span.RecordError(err)
            return nil, nfs.toNFSError(err)
        }
        
        if chunk := resp.GetChunk(); chunk != nil {
            data = append(data, chunk...)
        }
    }
    
    // Update access time in metadata
    nfs.updateAccessTime(ctx, handle.Path)
    
    span.SetAttributes(
        attribute.Int("bytes.read", len(data)),
    )
    
    return &nfsv4.ReadResponse{
        Data: data,
        EOF:  int64(len(data)) < req.Length,
    }, nil
}

func (nfs *NFSFileSystem) Write(ctx context.Context, req *nfsv4.WriteRequest) (*nfsv4.WriteResponse, error) {
    ctx, span := otel.Tracer("nfs-gateway").Start(ctx, "nfs.write",
        trace.WithAttributes(
            attribute.String("handle.id", req.Handle.String()),
            attribute.Int64("write.offset", req.Offset),
            attribute.Int("write.length", len(req.Data)),
        ),
    )
    defer span.End()
    
    handle := nfs.sessionMgr.GetHandle(req.Handle)
    if handle == nil {
        return nil, nfs.toNFSError(ErrBadHandle)
    }
    
    // For simplicity, we'll implement write-through semantics
    // In production, we'd want write-back caching for performance
    
    // Get current file
    currentFile, err := nfs.dfsClient.DownloadFile(ctx, &dfs.DownloadRequest{
        Path: handle.Path,
    })
    if err != nil && !errors.Is(err, ErrNotFound) {
        span.RecordError(err)
        return nil, nfs.toNFSError(err)
    }
    
    // Modify the file data
    modifiedData := nfs.applyWrite(currentFile, req.Offset, req.Data)
    
    // Upload modified file
    uploadReq := &dfs.UploadRequest{
        Path: handle.Path,
        Data: modifiedData,
        Metadata: map[string]string{
            "modified_by": "nfs_gateway",
            "nfs_handle":  req.Handle.String(),
        },
    }
    
    _, err = nfs.dfsClient.UploadFile(ctx, uploadReq)
    if err != nil {
        span.RecordError(err)
        return nil, nfs.toNFSError(err)
    }
    
    span.SetAttributes(
        attribute.Int("bytes.written", len(req.Data)),
    )
    
    return &nfsv4.WriteResponse{
        Count:  int32(len(req.Data)),
        Stable: nfsv4.FileSync,
    }, nil
}
```

### 2. SMB Server Gateway

#### Core Components
```go
type SMBGateway struct {
    server      *smb.Server
    fileSystem  FileSystemInterface
    authManager AuthenticationManager
    sessionMgr  *SessionManager
    treeConnect *TreeConnectionManager
    metrics     *SMBMetrics
}

// Implement SMB2 protocol operations
func (smb *SMBGateway) TreeConnect(ctx context.Context, req *smb2.TreeConnectRequest) (*smb2.TreeConnectResponse, error) {
    ctx, span := otel.Tracer("smb-gateway").Start(ctx, "smb.tree_connect",
        trace.WithAttributes(
            attribute.String("smb.operation", "tree_connect"),
            attribute.String("share.path", req.Path),
        ),
    )
    defer span.End()
    
    // Authenticate user
    user := smb.authManager.GetUserFromContext(ctx)
    if user == nil {
        return nil, smb.toSMBError(ErrAccessDenied)
    }
    
    // Map SMB share to DFS path
    dfsPath := smb.mapShareToDFSPath(req.Path)
    
    // Check permissions
    if !smb.authManager.HasPermission(user, dfsPath, "list") {
        span.RecordError(errors.New("permission denied"))
        return nil, smb.toSMBError(ErrAccessDenied)
    }
    
    // Create tree connection
    treeID := smb.treeConnect.CreateConnection(user.ID, dfsPath)
    
    span.SetAttributes(
        attribute.String("tree.id", treeID),
        attribute.String("dfs.path", dfsPath),
    )
    
    return &smb2.TreeConnectResponse{
        TreeID:      treeID,
        ShareType:   smb2.ShareTypeDisk,
        ShareFlags:  smb2.ShareFlagAccessBasedDirectoryEnumeration,
        Capabilities: smb2.ShareCapabilityDFS,
    }, nil
}

func (smb *SMBGateway) Create(ctx context.Context, req *smb2.CreateRequest) (*smb2.CreateResponse, error) {
    ctx, span := otel.Tracer("smb-gateway").Start(ctx, "smb.create",
        trace.WithAttributes(
            attribute.String("smb.operation", "create"),
            attribute.String("file.name", req.Name),
            attribute.Uint32("desired.access", uint32(req.DesiredAccess)),
        ),
    )
    defer span.End()
    
    // Get tree connection
    tree := smb.treeConnect.GetConnection(req.TreeID)
    if tree == nil {
        return nil, smb.toSMBError(ErrInvalidTreeID)
    }
    
    // Construct full DFS path
    dfsPath := filepath.Join(tree.DFSPath, req.Name)
    
    // Check if file exists
    fileInfo, err := smb.dfsClient.GetFileInfo(ctx, &dfs.GetFileInfoRequest{
        Path: dfsPath,
    })
    
    var fileID string
    if err != nil && errors.Is(err, ErrNotFound) {
        // Create new file
        if req.CreateDisposition == smb2.FileOpen {
            return nil, smb.toSMBError(ErrFileNotFound)
        }
        
        // Upload empty file
        uploadResp, err := smb.dfsClient.UploadFile(ctx, &dfs.UploadRequest{
            Path: dfsPath,
            Data: []byte{}, // Empty file
            Metadata: map[string]string{
                "created_by": "smb_gateway",
                "smb_tree_id": req.TreeID,
            },
        })
        if err != nil {
            span.RecordError(err)
            return nil, smb.toSMBError(err)
        }
        fileID = uploadResp.FileId
    } else if err != nil {
        span.RecordError(err)
        return nil, smb.toSMBError(err)
    } else {
        fileID = fileInfo.Id
    }
    
    // Create file handle
    handle := smb.sessionMgr.CreateSMBHandle(dfsPath, req.DesiredAccess)
    
    span.SetAttributes(
        attribute.String("file.id", fileID),
        attribute.String("handle.id", handle.ID),
    )
    
    return &smb2.CreateResponse{
        FileID:         handle.ToSMBFileID(),
        CreationTime:   fileInfo.CreatedAt,
        LastAccessTime: fileInfo.AccessedAt,
        LastWriteTime:  fileInfo.ModifiedAt,
        ChangeTime:     fileInfo.ModifiedAt,
        AllocationSize: fileInfo.Size,
        EndOfFile:      fileInfo.Size,
        FileAttributes: smb.toDFSAttributes(fileInfo),
    }, nil
}
```

### 3. Protocol Performance Optimizations

#### Caching Strategy
```go
type ProtocolCache struct {
    attrCache    *lru.Cache  // File attributes cache
    dataCache    *lru.Cache  // Small file data cache
    dirCache     *lru.Cache  // Directory listing cache
    negativeCache *lru.Cache  // Negative lookup cache
}

type CachedAttributes struct {
    FileInfo   *dfs.FileInfo
    CachedAt   time.Time
    TTL        time.Duration
}

func (pc *ProtocolCache) GetAttributes(path string) (*dfs.FileInfo, bool) {
    if cached, ok := pc.attrCache.Get(path); ok {
        attr := cached.(*CachedAttributes)
        if time.Since(attr.CachedAt) < attr.TTL {
            return attr.FileInfo, true
        }
        pc.attrCache.Remove(path) // Expired
    }
    return nil, false
}

func (pc *ProtocolCache) CacheAttributes(path string, info *dfs.FileInfo, ttl time.Duration) {
    pc.attrCache.Add(path, &CachedAttributes{
        FileInfo: info,
        CachedAt: time.Now(),
        TTL:      ttl,
    })
}
```

#### Write-Back Caching
```go
type WriteBackCache struct {
    dirty       map[string]*DirtyFile
    writeMutex  sync.RWMutex
    flushTicker *time.Ticker
    dfsClient   DFSClient
}

type DirtyFile struct {
    Path         string
    Data         []byte
    LastModified time.Time
    WriteCount   int
}

func (wbc *WriteBackCache) Write(path string, offset int64, data []byte) error {
    wbc.writeMutex.Lock()
    defer wbc.writeMutex.Unlock()
    
    if dirty, exists := wbc.dirty[path]; exists {
        // Merge with existing dirty data
        dirty.Data = wbc.mergeWrite(dirty.Data, offset, data)
        dirty.LastModified = time.Now()
        dirty.WriteCount++
    } else {
        // Load current file and apply write
        currentData := wbc.loadCurrentFile(path)
        wbc.dirty[path] = &DirtyFile{
            Path:         path,
            Data:         wbc.mergeWrite(currentData, offset, data),
            LastModified: time.Now(),
            WriteCount:   1,
        }
    }
    
    return nil
}

func (wbc *WriteBackCache) periodicFlush() {
    for range wbc.flushTicker.C {
        wbc.flushDirtyFiles()
    }
}
```

## Configuration and Deployment

### 1. Protocol Gateway Configuration

```yaml
# protocol-gateway-config.yaml
nfs:
  enabled: true
  version: "4.1"
  port: 2049
  exports:
    - path: "/dfs"
      dfs_path: "/"
      options: "rw,sync,no_root_squash"
      allowed_clients: ["10.0.0.0/8", "192.168.0.0/16"]
  
  performance:
    read_ahead_kb: 128
    write_back_cache: true
    attr_cache_ttl: "30s"
    dir_cache_ttl: "60s"

smb:
  enabled: true
  version: "3.1.1"
  port: 445
  shares:
    - name: "dfs"
      path: "/dfs"
      dfs_path: "/"
      read_only: false
      guest_access: false
      
  performance:
    signing_required: true
    encryption_required: false
    read_cache_size: "256MB"
    write_cache_size: "128MB"

auth:
  provider: "ldap"  # or "oauth2", "local"
  ldap:
    server: "ldap://ldap.example.com"
    base_dn: "ou=users,dc=example,dc=com"
    
cache:
  redis_cluster: ["redis-1:6379", "redis-2:6379", "redis-3:6379"]
  default_ttl: "5m"
  max_memory: "2GB"
```

### 2. Kubernetes Deployment

```yaml
# k8s/protocol-gateway.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dfs-protocol-gateway
  namespace: dfs-system
spec:
  replicas: 3
  selector:
    matchLabels:
      app: dfs-protocol-gateway
  template:
    metadata:
      labels:
        app: dfs-protocol-gateway
    spec:
      containers:
      - name: protocol-gateway
        image: dfs/protocol-gateway:latest
        ports:
        - containerPort: 2049
          name: nfs
          protocol: TCP
        - containerPort: 445
          name: smb
          protocol: TCP
        - containerPort: 8080
          name: metrics
          protocol: TCP
        env:
        - name: DFS_API_ENDPOINT
          value: "http://dfs-gateway:8080"
        - name: OTEL_EXPORTER_OTLP_ENDPOINT
          value: "http://otel-collector:4318"
        securityContext:
          privileged: true  # Required for NFS server
        volumeMounts:
        - name: config
          mountPath: /etc/dfs-protocol-gateway/
      volumes:
      - name: config
        configMap:
          name: protocol-gateway-config
---
apiVersion: v1
kind: Service
metadata:
  name: dfs-protocol-gateway
  namespace: dfs-system
spec:
  type: LoadBalancer
  ports:
  - port: 2049
    name: nfs
    protocol: TCP
  - port: 445
    name: smb
    protocol: TCP
  selector:
    app: dfs-protocol-gateway
```

## Benefits and Trade-offs

### Benefits
✅ **Enterprise Compatibility**: Works with existing NFS/SMB infrastructure
✅ **Seamless Migration**: Applications don't need modification
✅ **Storage Optimization**: Still get deduplication and compression benefits
✅ **Unified Management**: Single system for all access patterns
✅ **Performance**: Write-back caching optimizes protocol performance

### Trade-offs
⚠️ **Protocol Limitations**: NFS/SMB weren't designed for object storage semantics
⚠️ **Consistency Model**: May need to relax consistency for performance
⚠️ **Resource Overhead**: Protocol gateways require additional infrastructure
⚠️ **Security Complexity**: Managing multiple authentication mechanisms

### Performance Considerations
- **Caching**: Aggressive caching required for acceptable performance
- **Chunking**: Large files may have protocol overhead
- **Consistency**: May need to implement weaker consistency for protocols

## Recommendation

**Yes, NFS/SMB support makes sense** for our distributed file system because:

1. **Market Expansion**: Enables adoption by enterprises with existing infrastructure
2. **Migration Path**: Provides seamless migration from traditional NAS systems  
3. **Value Preservation**: Applications get storage optimization benefits transparently
4. **Competitive Advantage**: Becomes a drop-in replacement for expensive NAS solutions

The protocol gateway approach allows us to maintain our optimized backend while providing familiar interfaces to existing applications.

## Enterprise Migration Strategies

### Migration from Traditional NAS

#### 1. **Assessment Phase (2-4 weeks)**
```bash
# Network topology assessment
nmap -sn 10.0.0.0/16 | grep -i nas
showmount -e existing-nas-server

# Share inventory
smbclient -L existing-smb-server
mount -t nfs existing-nfs:/exports /mnt/assessment
find /mnt/assessment -type f -exec stat {} \; | awk '{print $8}' | sort -n
```

#### 2. **Parallel Deployment (4-6 weeks)**
```yaml
# migration-config.yaml
migration:
  source_nas:
    - server: "nas-01.corp.example.com"
      protocol: "nfs"
      exports: ["/home", "/shared", "/projects"]
    - server: "nas-02.corp.example.com" 
      protocol: "smb"
      shares: ["users$", "departments$", "archives$"]
      
  target_dfs:
    endpoint: "dfs-gateway.corp.example.com"
    migration_batch_size: "1TB"
    parallel_streams: 8
    
  cutover_schedule:
    - share: "/home"
      maintenance_window: "2024-03-15T02:00:00Z"
      rollback_deadline: "2024-03-15T06:00:00Z"
```

#### 3. **Data Synchronization Strategy**
```go
type MigrationManager struct {
    sourceNAS     []NASEndpoint
    targetDFS     DFSClient
    syncScheduler *CronScheduler
    metrics       *MigrationMetrics
}

func (mm *MigrationManager) StartIncrementalSync(ctx context.Context) {
    // Initial bulk copy
    mm.performBulkCopy(ctx)
    
    // Set up change detection
    for _, nas := range mm.sourceNAS {
        mm.setupChangeDetection(nas)
    }
    
    // Schedule incremental syncs every 15 minutes
    mm.syncScheduler.Schedule("*/15 * * * *", mm.incrementalSync)
}
```

### Protocol-Specific Migration Considerations

#### **NFS Migration Challenges**
- **UID/GID Mapping**: Maintain consistent user/group mappings
- **File Locking**: Handle NFSv4 advisory locks during migration
- **Access Patterns**: Monitor and optimize for common access patterns
- **Mount Point Compatibility**: Preserve existing mount point structures

#### **SMB Migration Challenges**
- **Domain Authentication**: Maintain AD/LDAP integration
- **Share Permissions**: Preserve ACLs and share-level permissions
- **Opportunistic Locking**: Handle SMB oplock behavior changes
- **DFS Referrals**: Update DFS namespace to point to new system

### Protocol Performance Tuning

#### **NFS Performance Optimization**
```yaml
nfs_tuning:
  server_side:
    nfsd_threads: 64
    readahead_kb: 256
    write_cache_size: "512MB"
    attr_cache_ttl: "60s"
    
  client_side:
    rsize: 1048576    # 1MB read size
    wsize: 1048576    # 1MB write size
    tcp: true
    vers: "4.1"
    proto: "tcp"
    
  mount_options: "rw,tcp,vers=4.1,rsize=1048576,wsize=1048576,timeo=600"
```

#### **SMB Performance Optimization**
```yaml
smb_tuning:
  server_side:
    max_connections: 1000
    socket_options: "TCP_NODELAY IPTOS_LOWDELAY"
    read_ahead_size: "1MB"
    write_cache_size: "256MB"
    
  protocol_settings:
    smb_encrypt: "required"  # For sensitive data
    smb_signing: "required"
    smb2_leases: true       # Enable SMB2 leases for better caching
    
  performance:
    large_readwrite: true
    unix_extensions: false  # Disable for better Windows compatibility
```

## Legacy Application Compatibility

### **Legacy Application Support Matrix**

| Application Type | NFS Compatibility | SMB Compatibility | Migration Effort |
|-----------------|------------------|------------------|-----------------|
| CAD/CAM Software | ✅ Full | ⚠️ Limited | Low |
| Database Backups | ✅ Full | ✅ Full | Low |
| Media Workflows | ✅ Full | ⚠️ Performance | Medium |
| Home Directories | ✅ Full | ✅ Full | Low |
| Build Systems | ✅ Full | ❌ Not Recommended | Low-Medium |
| Archive Systems | ✅ Full | ✅ Full | Low |

### **Application-Specific Configurations**

#### **CAD/CAM Workstations**
```yaml
# optimized for large file operations
nfs_cad_profile:
  mount_options: "rw,tcp,vers=4.1,rsize=8388608,wsize=8388608,timeo=600,retrans=2"
  cache_settings:
    attr_cache: "300s"    # Longer cache for large files
    dir_cache: "600s"
    read_ahead: "16MB"
```

#### **Media Production Workflows**
```yaml
# optimized for sequential access patterns
nfs_media_profile:
  mount_options: "rw,tcp,vers=4.1,rsize=4194304,wsize=4194304,ac,lookupcache=positive"
  storage_tier: "hot"     # Keep media files in hot tier
  compression: "lz4"      # Fast compression for media files
```

## Monitoring and Alerting for Protocol Gateways

### **Protocol-Specific Metrics**
```go
type ProtocolMetrics struct {
    // NFS metrics
    NFSOperationsTotal     prometheus.CounterVec
    NFSOperationDuration   prometheus.HistogramVec
    NFSActiveConnections   prometheus.Gauge
    NFSCacheHitRate       prometheus.Gauge
    
    // SMB metrics  
    SMBSessionsActive      prometheus.Gauge
    SMBTreeConnections     prometheus.Gauge
    SMBOpLockBreaks       prometheus.Counter
    SMBAuthentication     prometheus.CounterVec
    
    // Common metrics
    ProtocolErrorRate     prometheus.CounterVec
    BackendLatency        prometheus.HistogramVec
    CacheMemoryUsage      prometheus.Gauge
}
```

### **Alert Rules**
```yaml
# alerts/protocol-gateway.yml
groups:
- name: protocol-gateway
  rules:
  - alert: NFSHighErrorRate
    expr: rate(nfs_errors_total[5m]) > 0.05
    for: 2m
    annotations:
      summary: "High NFS error rate detected"
      
  - alert: SMBAuthenticationFailures
    expr: rate(smb_auth_failures_total[5m]) > 10
    for: 1m
    annotations:
      summary: "High SMB authentication failure rate"
      
  - alert: ProtocolGatewayCacheEviction
    expr: rate(protocol_cache_evictions_total[5m]) > 100
    for: 5m
    annotations:
      summary: "High cache eviction rate - consider increasing cache size"
```
