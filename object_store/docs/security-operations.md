# Security and Operations Guide

## Table of Contents
1. [Security Architecture](#security-architecture)
2. [Authentication and Authorization](#authentication-and-authorization)
3. [Data Encryption](#data-encryption)
4. [Network Security](#network-security)
5. [Operational Procedures](#operational-procedures)
6. [Incident Response](#incident-response)
7. [Compliance and Auditing](#compliance-and-auditing)
8. [Security Monitoring](#security-monitoring)

## Security Architecture

### Defense in Depth

The distributed object store implements multiple layers of security:

```
┌─────────────────────────────────────────────────────────────┐
│                    External Threats                         │
└─────────────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────────────┐
│  Layer 1: Network Perimeter (WAF, DDoS Protection)         │
└─────────────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────────────┐
│  Layer 2: API Gateway (Rate Limiting, Auth)                │
└─────────────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────────────┐
│  Layer 3: Application Security (RBAC, Input Validation)    │
└─────────────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────────────┐
│  Layer 4: Data Security (Encryption, Access Control)       │
└─────────────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────────────┐
│  Layer 5: Infrastructure Security (VPC, IAM, HSM)          │
└─────────────────────────────────────────────────────────────┘
```

### Security Principles

1. **Zero Trust**: Never trust, always verify
2. **Least Privilege**: Minimum required permissions
3. **Defense in Depth**: Multiple security layers
4. **Encryption Everywhere**: Data protection at rest and in transit
5. **Continuous Monitoring**: Real-time threat detection

## Authentication and Authorization

### Multi-Factor Authentication

```go
package auth

type AuthenticationService struct {
    jwtManager    *JWTManager
    mfaProvider   *MFAProvider
    userStore     *UserStore
    auditLogger   *AuditLogger
}

type AuthRequest struct {
    Username    string `json:"username"`
    Password    string `json:"password"`
    MFAToken    string `json:"mfa_token,omitempty"`
    APIKey      string `json:"api_key,omitempty"`
    ClientCert  []byte `json:"client_cert,omitempty"`
}

type AuthResponse struct {
    AccessToken  string    `json:"access_token"`
    RefreshToken string    `json:"refresh_token"`
    ExpiresIn    int       `json:"expires_in"`
    Scope        []string  `json:"scope"`
    UserID       string    `json:"user_id"`
}

func (as *AuthenticationService) Authenticate(ctx context.Context, req *AuthRequest) (*AuthResponse, error) {
    // Step 1: Validate primary credentials
    user, err := as.validateCredentials(ctx, req.Username, req.Password)
    if err != nil {
        as.auditLogger.LogFailedAuth(req.Username, "invalid_credentials", getClientIP(ctx))
        return nil, ErrInvalidCredentials
    }
    
    // Step 2: Validate MFA if required
    if user.MFARequired {
        if req.MFAToken == "" {
            return nil, ErrMFARequired
        }
        
        if !as.mfaProvider.ValidateToken(user.ID, req.MFAToken) {
            as.auditLogger.LogFailedAuth(req.Username, "invalid_mfa", getClientIP(ctx))
            return nil, ErrInvalidMFA
        }
    }
    
    // Step 3: Generate tokens
    accessToken, err := as.jwtManager.GenerateAccessToken(user)
    if err != nil {
        return nil, err
    }
    
    refreshToken, err := as.jwtManager.GenerateRefreshToken(user)
    if err != nil {
        return nil, err
    }
    
    // Step 4: Log successful authentication
    as.auditLogger.LogSuccessfulAuth(user.ID, getClientIP(ctx))
    
    return &AuthResponse{
        AccessToken:  accessToken,
        RefreshToken: refreshToken,
        ExpiresIn:    3600, // 1 hour
        Scope:        user.Permissions,
        UserID:       user.ID,
    }, nil
}
```

### Role-Based Access Control (RBAC)

```go
type Permission struct {
    Resource string   `json:"resource"`
    Actions  []string `json:"actions"`
    Effect   string   `json:"effect"` // "allow" or "deny"
}

type Role struct {
    Name        string       `json:"name"`
    Description string       `json:"description"`
    Permissions []Permission `json:"permissions"`
}

type User struct {
    ID          string   `json:"id"`
    Username    string   `json:"username"`
    Email       string   `json:"email"`
    Roles       []string `json:"roles"`
    MFARequired bool     `json:"mfa_required"`
    Status      string   `json:"status"`
}

var PredefinedRoles = map[string]Role{
    "admin": {
        Name: "Administrator",
        Permissions: []Permission{
            {Resource: "*", Actions: []string{"*"}, Effect: "allow"},
        },
    },
    "bucket_owner": {
        Name: "Bucket Owner",
        Permissions: []Permission{
            {Resource: "bucket:*", Actions: []string{"create", "delete", "read", "write"}, Effect: "allow"},
            {Resource: "object:*", Actions: []string{"create", "delete", "read", "write"}, Effect: "allow"},
        },
    },
    "read_only": {
        Name: "Read Only",
        Permissions: []Permission{
            {Resource: "bucket:*", Actions: []string{"read"}, Effect: "allow"},
            {Resource: "object:*", Actions: []string{"read"}, Effect: "allow"},
        },
    },
    "upload_only": {
        Name: "Upload Only",
        Permissions: []Permission{
            {Resource: "object:*", Actions: []string{"create", "write"}, Effect: "allow"},
        },
    },
}

func (as *AuthenticationService) CheckPermission(ctx context.Context, userID, resource, action string) error {
    user, err := as.userStore.GetUser(ctx, userID)
    if err != nil {
        return err
    }
    
    // Check each role's permissions
    for _, roleName := range user.Roles {
        role, exists := PredefinedRoles[roleName]
        if !exists {
            continue
        }
        
        for _, permission := range role.Permissions {
            if as.matchesResource(permission.Resource, resource) &&
               as.matchesAction(permission.Actions, action) {
                
                if permission.Effect == "allow" {
                    return nil
                } else if permission.Effect == "deny" {
                    return ErrAccessDenied
                }
            }
        }
    }
    
    return ErrAccessDenied
}
```

### API Key Management

```go
type APIKey struct {
    KeyID       string    `json:"key_id"`
    UserID      string    `json:"user_id"`
    Name        string    `json:"name"`
    Prefix      string    `json:"prefix"`
    Hash        string    `json:"hash"`
    Permissions []string  `json:"permissions"`
    CreatedAt   time.Time `json:"created_at"`
    ExpiresAt   *time.Time `json:"expires_at,omitempty"`
    LastUsed    *time.Time `json:"last_used,omitempty"`
    Status      string    `json:"status"`
}

func (as *AuthenticationService) CreateAPIKey(ctx context.Context, userID, name string, permissions []string, ttl *time.Duration) (*APIKey, string, error) {
    // Generate cryptographically secure key
    keyBytes := make([]byte, 32)
    if _, err := rand.Read(keyBytes); err != nil {
        return nil, "", err
    }
    
    keyValue := base64.URLEncoding.EncodeToString(keyBytes)
    prefix := keyValue[:8]
    hash := sha256.Sum256([]byte(keyValue))
    
    var expiresAt *time.Time
    if ttl != nil {
        exp := time.Now().Add(*ttl)
        expiresAt = &exp
    }
    
    apiKey := &APIKey{
        KeyID:       uuid.New().String(),
        UserID:      userID,
        Name:        name,
        Prefix:      prefix,
        Hash:        hex.EncodeToString(hash[:]),
        Permissions: permissions,
        CreatedAt:   time.Now(),
        ExpiresAt:   expiresAt,
        Status:      "active",
    }
    
    if err := as.userStore.CreateAPIKey(ctx, apiKey); err != nil {
        return nil, "", err
    }
    
    return apiKey, keyValue, nil
}
```

## Data Encryption

### Encryption at Rest

```go
package encryption

type EncryptionManager struct {
    kmsClient    *KMSClient
    localKeys    map[string][]byte
    keyRotation  *KeyRotationManager
}

type EncryptionConfig struct {
    Algorithm       string `json:"algorithm"`        // AES-256-GCM
    KeySize         int    `json:"key_size"`         // 256 bits
    BlockSize       int    `json:"block_size"`       // 16 bytes
    RotationPeriod  string `json:"rotation_period"`  // 90 days
}

func NewEncryptionManager(kmsEndpoint string) *EncryptionManager {
    return &EncryptionManager{
        kmsClient:   NewKMSClient(kmsEndpoint),
        localKeys:   make(map[string][]byte),
        keyRotation: NewKeyRotationManager(),
    }
}

func (em *EncryptionManager) EncryptObject(ctx context.Context, objectID string, data []byte) ([]byte, error) {
    // Get or create data encryption key (DEK)
    dek, err := em.getDataEncryptionKey(ctx, objectID)
    if err != nil {
        return nil, err
    }
    
    // Create cipher
    block, err := aes.NewCipher(dek)
    if err != nil {
        return nil, err
    }
    
    gcm, err := cipher.NewGCM(block)
    if err != nil {
        return nil, err
    }
    
    // Generate random nonce
    nonce := make([]byte, gcm.NonceSize())
    if _, err := io.ReadFull(rand.Reader, nonce); err != nil {
        return nil, err
    }
    
    // Encrypt data
    ciphertext := gcm.Seal(nonce, nonce, data, nil)
    
    // Prepend metadata
    metadata := EncryptionMetadata{
        Algorithm: "AES-256-GCM",
        KeyID:     em.getKeyID(objectID),
        Version:   1,
    }
    
    metadataBytes, _ := json.Marshal(metadata)
    result := append(metadataBytes, ciphertext...)
    
    return result, nil
}

func (em *EncryptionManager) DecryptObject(ctx context.Context, encryptedData []byte) ([]byte, error) {
    // Extract metadata
    var metadata EncryptionMetadata
    metadataLen := binary.BigEndian.Uint32(encryptedData[:4])
    
    if err := json.Unmarshal(encryptedData[4:4+metadataLen], &metadata); err != nil {
        return nil, err
    }
    
    ciphertext := encryptedData[4+metadataLen:]
    
    // Get decryption key
    dek, err := em.getDataEncryptionKey(ctx, metadata.KeyID)
    if err != nil {
        return nil, err
    }
    
    // Decrypt
    block, err := aes.NewCipher(dek)
    if err != nil {
        return nil, err
    }
    
    gcm, err := cipher.NewGCM(block)
    if err != nil {
        return nil, err
    }
    
    nonceSize := gcm.NonceSize()
    nonce, ciphertext := ciphertext[:nonceSize], ciphertext[nonceSize:]
    
    plaintext, err := gcm.Open(nil, nonce, ciphertext, nil)
    if err != nil {
        return nil, err
    }
    
    return plaintext, nil
}
```

### Key Management

```go
type KMSClient struct {
    endpoint     string
    credentials  *AWSCredentials
    client       *aws.Client
}

type DataEncryptionKey struct {
    KeyID      string    `json:"key_id"`
    Plaintext  []byte    `json:"-"`              // Never logged
    Ciphertext []byte    `json:"ciphertext"`     // Encrypted by KEK
    CreatedAt  time.Time `json:"created_at"`
    Algorithm  string    `json:"algorithm"`
}

func (kms *KMSClient) GenerateDataKey(ctx context.Context, keyID string) (*DataEncryptionKey, error) {
    // Generate random key
    key := make([]byte, 32) // 256 bits
    if _, err := rand.Read(key); err != nil {
        return nil, err
    }
    
    // Encrypt with master key
    encryptedKey, err := kms.encryptWithMasterKey(ctx, keyID, key)
    if err != nil {
        return nil, err
    }
    
    return &DataEncryptionKey{
        KeyID:      uuid.New().String(),
        Plaintext:  key,
        Ciphertext: encryptedKey,
        CreatedAt:  time.Now(),
        Algorithm:  "AES-256",
    }, nil
}

func (kms *KMSClient) RotateKeys(ctx context.Context) error {
    // Get all active keys
    keys, err := kms.listActiveKeys(ctx)
    if err != nil {
        return err
    }
    
    for _, key := range keys {
        if time.Since(key.CreatedAt) > 90*24*time.Hour { // 90 days
            if err := kms.rotateKey(ctx, key); err != nil {
                log.Printf("Failed to rotate key %s: %v", key.KeyID, err)
                continue
            }
        }
    }
    
    return nil
}
```

### Encryption in Transit

```go
type TLSConfig struct {
    MinVersion    uint16
    CipherSuites  []uint16
    Certificates  []tls.Certificate
    ClientAuth    tls.ClientAuthType
}

func NewSecureTLSConfig() *tls.Config {
    return &tls.Config{
        MinVersion: tls.VersionTLS13,
        CipherSuites: []uint16{
            tls.TLS_AES_256_GCM_SHA384,
            tls.TLS_CHACHA20_POLY1305_SHA256,
            tls.TLS_AES_128_GCM_SHA256,
        },
        PreferServerCipherSuites: true,
        CurvePreferences: []tls.CurveID{
            tls.X25519,
            tls.CurveP384,
            tls.CurveP256,
        },
        ClientAuth: tls.RequireAndVerifyClientCert,
    }
}
```

## Network Security

### Web Application Firewall (WAF)

```yaml
waf_rules:
  # Rate limiting rules
  - name: "api_rate_limit"
    priority: 1
    condition: "rate(5m) > 1000"
    action: "block"
    duration: "1h"
    
  # Geographic restrictions
  - name: "geo_blocking"
    priority: 2
    condition: "country in ['CN', 'RU', 'KP']"
    action: "block"
    exceptions: ["whitelist_ips"]
    
  # SQL injection protection
  - name: "sqli_protection"
    priority: 3
    condition: "body contains sql_injection_patterns"
    action: "block"
    log_level: "critical"
    
  # DDoS protection
  - name: "ddos_protection"
    priority: 4
    condition: "concurrent_connections > 10000"
    action: "challenge"
    challenge_type: "captcha"
```

### VPC and Network Isolation

```yaml
network_architecture:
  vpc:
    cidr: "10.0.0.0/16"
    enable_dns_hostnames: true
    enable_dns_support: true
    
  subnets:
    public:
      - cidr: "10.0.1.0/24"
        availability_zone: "us-west-1a"
      - cidr: "10.0.2.0/24"
        availability_zone: "us-west-1b"
        
    private:
      - cidr: "10.0.10.0/24"
        availability_zone: "us-west-1a"
      - cidr: "10.0.11.0/24" 
        availability_zone: "us-west-1b"
        
    database:
      - cidr: "10.0.20.0/24"
        availability_zone: "us-west-1a"
      - cidr: "10.0.21.0/24"
        availability_zone: "us-west-1b"
        
  security_groups:
    api_gateway:
      ingress:
        - port: 443
          protocol: "tcp"
          source: "0.0.0.0/0"
        - port: 80
          protocol: "tcp"
          source: "0.0.0.0/0"
          
    storage_nodes:
      ingress:
        - port: 8080
          protocol: "tcp"
          source: "api_gateway_sg"
        - port: 22
          protocol: "tcp"
          source: "bastion_sg"
          
    metadata_db:
      ingress:
        - port: 5432
          protocol: "tcp"
          source: "api_gateway_sg"
```

## Operational Procedures

### Deployment Procedures

```bash
#!/bin/bash
# Secure deployment script

set -euo pipefail

# Configuration
ENVIRONMENT=${1:-staging}
VERSION=${2:-latest}
NAMESPACE="objectstore-${ENVIRONMENT}"

# Pre-deployment checks
echo "Running pre-deployment security checks..."

# 1. Verify image signatures
docker trust inspect "${IMAGE_REGISTRY}/objectstore:${VERSION}"

# 2. Scan for vulnerabilities
trivy image --exit-code 1 --severity HIGH,CRITICAL "${IMAGE_REGISTRY}/objectstore:${VERSION}"

# 3. Validate Kubernetes manifests
kubectl apply --dry-run=client -f manifests/

# 4. Check RBAC permissions
kubectl auth can-i create pods --namespace="${NAMESPACE}"

# Deployment with rolling update
echo "Starting rolling deployment..."
kubectl set image deployment/objectstore-api \
    objectstore="${IMAGE_REGISTRY}/objectstore:${VERSION}" \
    --namespace="${NAMESPACE}"

# Wait for rollout completion
kubectl rollout status deployment/objectstore-api --namespace="${NAMESPACE}" --timeout=300s

# Post-deployment verification
echo "Running post-deployment checks..."

# Health check
for i in {1..30}; do
    if curl -f "https://api.${ENVIRONMENT}.objectstore.com/health"; then
        echo "Health check passed"
        break
    fi
    sleep 10
done

# Security validation
echo "Running security validation..."
./scripts/security-tests.sh "${ENVIRONMENT}"

echo "Deployment completed successfully"
```

### Backup and Recovery

```go
package backup

type BackupManager struct {
    storageClient *StorageClient
    metadataDB    *Database
    scheduler     *CronScheduler
    config        *BackupConfig
}

type BackupConfig struct {
    RetentionPeriod  time.Duration
    CompressionLevel int
    EncryptionKey    string
    CrossRegion      bool
    Frequency        string // cron expression
}

func (bm *BackupManager) CreateBackup(ctx context.Context, backupType string) (*Backup, error) {
    backup := &Backup{
        ID:        uuid.New().String(),
        Type:      backupType,
        StartTime: time.Now(),
        Status:    "in_progress",
    }
    
    switch backupType {
    case "metadata":
        return bm.backupMetadata(ctx, backup)
    case "full":
        return bm.backupFull(ctx, backup)
    case "incremental":
        return bm.backupIncremental(ctx, backup)
    default:
        return nil, fmt.Errorf("unknown backup type: %s", backupType)
    }
}

func (bm *BackupManager) RestoreBackup(ctx context.Context, backupID string, restorePoint time.Time) error {
    backup, err := bm.getBackup(ctx, backupID)
    if err != nil {
        return err
    }
    
    // Verify backup integrity
    if err := bm.verifyBackupIntegrity(ctx, backup); err != nil {
        return fmt.Errorf("backup integrity check failed: %w", err)
    }
    
    // Create restore point
    if err := bm.createRestorePoint(ctx); err != nil {
        return err
    }
    
    // Perform restore
    switch backup.Type {
    case "metadata":
        return bm.restoreMetadata(ctx, backup, restorePoint)
    case "full":
        return bm.restoreFull(ctx, backup, restorePoint)
    case "incremental":
        return bm.restoreIncremental(ctx, backup, restorePoint)
    }
    
    return nil
}
```

### Maintenance Windows

```go
type MaintenanceWindow struct {
    ID          string           `json:"id"`
    Title       string           `json:"title"`
    Description string           `json:"description"`
    StartTime   time.Time        `json:"start_time"`
    Duration    time.Duration    `json:"duration"`
    Impact      ImpactLevel      `json:"impact"`
    Procedures  []Procedure      `json:"procedures"`
    Approvals   []Approval       `json:"approvals"`
    Status      MaintenanceStatus `json:"status"`
}

type ImpactLevel string

const (
    ImpactNone   ImpactLevel = "none"
    ImpactLow    ImpactLevel = "low"
    ImpactMedium ImpactLevel = "medium"
    ImpactHigh   ImpactLevel = "high"
)

func (mw *MaintenanceWindow) Execute(ctx context.Context) error {
    // Pre-maintenance checks
    if err := mw.runPreChecks(ctx); err != nil {
        return fmt.Errorf("pre-checks failed: %w", err)
    }
    
    // Set system to maintenance mode
    if err := mw.enableMaintenanceMode(ctx); err != nil {
        return err
    }
    defer mw.disableMaintenanceMode(ctx)
    
    // Execute procedures
    for _, procedure := range mw.Procedures {
        if err := mw.executeProcedure(ctx, procedure); err != nil {
            // Attempt rollback
            mw.rollback(ctx, procedure)
            return fmt.Errorf("procedure %s failed: %w", procedure.Name, err)
        }
    }
    
    // Post-maintenance validation
    return mw.runPostChecks(ctx)
}
```

## Incident Response

### Security Incident Classification

```go
type SecurityIncident struct {
    ID           string           `json:"id"`
    Title        string           `json:"title"`
    Description  string           `json:"description"`
    Severity     SeverityLevel    `json:"severity"`
    Category     IncidentCategory `json:"category"`
    DetectedAt   time.Time        `json:"detected_at"`
    ReportedBy   string           `json:"reported_by"`
    Assignee     string           `json:"assignee"`
    Status       IncidentStatus   `json:"status"`
    Timeline     []TimelineEvent  `json:"timeline"`
    Artifacts    []Artifact       `json:"artifacts"`
    Impact       ImpactAssessment `json:"impact"`
}

type SeverityLevel string

const (
    SeverityCritical SeverityLevel = "critical"  // Data breach, system compromise
    SeverityHigh     SeverityLevel = "high"      // Failed authentication attempts
    SeverityMedium   SeverityLevel = "medium"    // Policy violations
    SeverityLow      SeverityLevel = "low"       // Informational
)

type IncidentCategory string

const (
    CategoryDataBreach     IncidentCategory = "data_breach"
    CategoryUnauthorized   IncidentCategory = "unauthorized_access"
    CategoryMalware        IncidentCategory = "malware"
    CategoryDDoS           IncidentCategory = "ddos"
    CategoryDataLoss       IncidentCategory = "data_loss"
    CategoryPolicyViolation IncidentCategory = "policy_violation"
)
```

### Automated Response Actions

```go
type IncidentResponse struct {
    detector      *ThreatDetector
    responder     *AutoResponder
    notifier      *NotificationService
    forensics     *ForensicsCollector
}

func (ir *IncidentResponse) HandleIncident(ctx context.Context, incident *SecurityIncident) error {
    // 1. Immediate containment
    if err := ir.containThreat(ctx, incident); err != nil {
        log.Printf("Failed to contain threat: %v", err)
    }
    
    // 2. Evidence collection
    go ir.collectEvidence(ctx, incident)
    
    // 3. Notification
    if err := ir.notifyStakeholders(ctx, incident); err != nil {
        log.Printf("Failed to send notifications: %v", err)
    }
    
    // 4. Automated response
    return ir.executeResponsePlan(ctx, incident)
}

func (ir *IncidentResponse) containThreat(ctx context.Context, incident *SecurityIncident) error {
    switch incident.Category {
    case CategoryUnauthorized:
        // Disable compromised accounts
        return ir.disableCompromisedAccounts(ctx, incident)
        
    case CategoryDDoS:
        // Enable DDoS protection
        return ir.enableDDoSProtection(ctx, incident)
        
    case CategoryDataBreach:
        // Isolate affected systems
        return ir.isolateAffectedSystems(ctx, incident)
        
    default:
        // Generic containment
        return ir.increaseSecurityPosture(ctx, incident)
    }
}
```

### Forensics and Evidence Collection

```go
type ForensicsCollector struct {
    storage     *SecureStorage
    chain       *ChainOfCustody
    tools       *ForensicsTools
}

type Evidence struct {
    ID          string    `json:"id"`
    Type        string    `json:"type"`
    Source      string    `json:"source"`
    CollectedAt time.Time `json:"collected_at"`
    CollectedBy string    `json:"collected_by"`
    Hash        string    `json:"hash"`
    Size        int64     `json:"size"`
    Metadata    map[string]interface{} `json:"metadata"`
}

func (fc *ForensicsCollector) CollectEvidence(ctx context.Context, incident *SecurityIncident) error {
    evidence := make([]*Evidence, 0)
    
    // Collect logs
    logs, err := fc.collectLogs(ctx, incident.DetectedAt.Add(-1*time.Hour), time.Now())
    if err == nil {
        evidence = append(evidence, logs...)
    }
    
    // Collect network traffic
    network, err := fc.collectNetworkTraffic(ctx, incident.DetectedAt.Add(-30*time.Minute), time.Now())
    if err == nil {
        evidence = append(evidence, network...)
    }
    
    // Collect system state
    system, err := fc.collectSystemState(ctx)
    if err == nil {
        evidence = append(evidence, system...)
    }
    
    // Store evidence securely
    for _, e := range evidence {
        if err := fc.storage.StoreEvidence(ctx, e); err != nil {
            log.Printf("Failed to store evidence %s: %v", e.ID, err)
        }
        
        // Update chain of custody
        fc.chain.AddEntry(e.ID, "collected", time.Now(), "system")
    }
    
    return nil
}
```

## Compliance and Auditing

### Audit Logging

```go
type AuditLogger struct {
    writer    io.Writer
    formatter AuditFormatter
    filter    AuditFilter
}

type AuditEvent struct {
    EventID     string                 `json:"event_id"`
    Timestamp   time.Time              `json:"timestamp"`
    EventType   string                 `json:"event_type"`
    UserID      string                 `json:"user_id,omitempty"`
    SessionID   string                 `json:"session_id,omitempty"`
    SourceIP    string                 `json:"source_ip"`
    UserAgent   string                 `json:"user_agent,omitempty"`
    Resource    string                 `json:"resource"`
    Action      string                 `json:"action"`
    Result      string                 `json:"result"`
    Metadata    map[string]interface{} `json:"metadata,omitempty"`
    Severity    string                 `json:"severity"`
}

func (al *AuditLogger) LogEvent(event *AuditEvent) {
    if al.filter.ShouldLog(event) {
        formatted := al.formatter.Format(event)
        al.writer.Write(formatted)
    }
}

// Predefined audit events
func (al *AuditLogger) LogLogin(userID, sourceIP string, success bool) {
    result := "success"
    if !success {
        result = "failure"
    }
    
    al.LogEvent(&AuditEvent{
        EventID:   uuid.New().String(),
        Timestamp: time.Now(),
        EventType: "authentication",
        UserID:    userID,
        SourceIP:  sourceIP,
        Action:    "login",
        Result:    result,
        Severity:  "info",
    })
}

func (al *AuditLogger) LogDataAccess(userID, resource, action string, success bool) {
    result := "success"
    severity := "info"
    if !success {
        result = "failure"
        severity = "warning"
    }
    
    al.LogEvent(&AuditEvent{
        EventID:   uuid.New().String(),
        Timestamp: time.Now(),
        EventType: "data_access",
        UserID:    userID,
        Resource:  resource,
        Action:    action,
        Result:    result,
        Severity:  severity,
    })
}
```

### Compliance Reporting

```go
type ComplianceReporter struct {
    auditStore *AuditStore
    templates  map[string]*ReportTemplate
}

type ComplianceReport struct {
    ReportID     string                 `json:"report_id"`
    Type         string                 `json:"type"`
    Period       Period                 `json:"period"`
    GeneratedAt  time.Time              `json:"generated_at"`
    GeneratedBy  string                 `json:"generated_by"`
    Sections     []ReportSection        `json:"sections"`
    Findings     []Finding              `json:"findings"`
    Recommendations []Recommendation    `json:"recommendations"`
}

type Finding struct {
    ID          string `json:"id"`
    Type        string `json:"type"`
    Severity    string `json:"severity"`
    Description string `json:"description"`
    Evidence    []string `json:"evidence"`
    Remediation string `json:"remediation"`
}

func (cr *ComplianceReporter) GenerateSOC2Report(ctx context.Context, period Period) (*ComplianceReport, error) {
    report := &ComplianceReport{
        ReportID:    uuid.New().String(),
        Type:        "SOC2",
        Period:      period,
        GeneratedAt: time.Now(),
    }
    
    // Security controls assessment
    securityFindings, err := cr.assessSecurityControls(ctx, period)
    if err != nil {
        return nil, err
    }
    report.Findings = append(report.Findings, securityFindings...)
    
    // Availability assessment
    availabilityFindings, err := cr.assessAvailability(ctx, period)
    if err != nil {
        return nil, err
    }
    report.Findings = append(report.Findings, availabilityFindings...)
    
    // Privacy assessment
    privacyFindings, err := cr.assessPrivacy(ctx, period)
    if err != nil {
        return nil, err
    }
    report.Findings = append(report.Findings, privacyFindings...)
    
    return report, nil
}
```

## Security Monitoring

### Real-time Threat Detection

```go
type ThreatDetector struct {
    rules        []DetectionRule
    correlator   *EventCorrelator
    mlDetector   *MLAnomalyDetector
    alertManager *AlertManager
}

type DetectionRule struct {
    ID          string        `json:"id"`
    Name        string        `json:"name"`
    Description string        `json:"description"`
    Severity    string        `json:"severity"`
    Query       string        `json:"query"`
    Threshold   int           `json:"threshold"`
    TimeWindow  time.Duration `json:"time_window"`
    Enabled     bool          `json:"enabled"`
}

var SecurityRules = []DetectionRule{
    {
        ID:          "failed_auth_threshold",
        Name:        "Multiple Failed Authentication Attempts",
        Description: "Detects multiple failed authentication attempts from same IP",
        Severity:    "medium",
        Query:       "event_type:authentication AND result:failure",
        Threshold:   5,
        TimeWindow:  5 * time.Minute,
        Enabled:     true,
    },
    {
        ID:          "data_exfiltration",
        Name:        "Unusual Data Download Volume",
        Description: "Detects unusual data download patterns",
        Severity:    "high",
        Query:       "action:download",
        Threshold:   1000, // MB
        TimeWindow:  1 * time.Hour,
        Enabled:     true,
    },
    {
        ID:          "privilege_escalation",
        Name:        "Privilege Escalation Attempt",
        Description: "Detects attempts to access restricted resources",
        Severity:    "critical",
        Query:       "result:failure AND action:admin",
        Threshold:   1,
        TimeWindow:  1 * time.Minute,
        Enabled:     true,
    },
}

func (td *ThreatDetector) ProcessEvent(ctx context.Context, event *AuditEvent) {
    // Apply detection rules
    for _, rule := range td.rules {
        if !rule.Enabled {
            continue
        }
        
        if td.matchesRule(event, rule) {
            count := td.getEventCount(ctx, rule.Query, rule.TimeWindow)
            if count >= rule.Threshold {
                alert := &SecurityAlert{
                    ID:        uuid.New().String(),
                    RuleID:    rule.ID,
                    Severity:  rule.Severity,
                    Timestamp: time.Now(),
                    Event:     event,
                    Count:     count,
                }
                
                td.alertManager.SendAlert(ctx, alert)
            }
        }
    }
    
    // Machine learning detection
    if anomaly := td.mlDetector.DetectAnomaly(event); anomaly != nil {
        alert := &SecurityAlert{
            ID:        uuid.New().String(),
            RuleID:    "ml_anomaly",
            Severity:  anomaly.Severity,
            Timestamp: time.Now(),
            Event:     event,
            Metadata:  anomaly.Metadata,
        }
        
        td.alertManager.SendAlert(ctx, alert)
    }
}
```

This comprehensive security and operations guide provides the framework for maintaining a secure, compliant, and operationally robust distributed object store system.
