# Security and Operations for Photo Storage System

## Security Architecture

### Security Layers

```
┌─────────────────────────────────────┐
│            User Layer               │
│  Authentication & Authorization     │
├─────────────────────────────────────┤
│          Network Layer              │
│   WAF, DDoS Protection, TLS        │
├─────────────────────────────────────┤
│        Application Layer            │
│  Input Validation, Rate Limiting    │
├─────────────────────────────────────┤
│           Data Layer                │
│  Encryption, Access Control        │
├─────────────────────────────────────┤
│       Infrastructure Layer          │
│  Network Security, Monitoring      │
└─────────────────────────────────────┘
```

## Authentication and Authorization

### Multi-Factor Authentication (MFA)
```go
type AuthenticationService struct {
    userStore     UserStore
    mfaProvider   MFAProvider
    sessionStore  SessionStore
    jwtManager    JWTManager
}

type AuthRequest struct {
    Email      string `json:"email"`
    Password   string `json:"password"`
    MFACode    string `json:"mfa_code,omitempty"`
    DeviceID   string `json:"device_id"`
    UserAgent  string `json:"user_agent"`
    IPAddress  string `json:"ip_address"`
}

func (as *AuthenticationService) Authenticate(req *AuthRequest) (*AuthResponse, error) {
    // Step 1: Verify credentials
    user, err := as.userStore.ValidateCredentials(req.Email, req.Password)
    if err != nil {
        as.logFailedAttempt(req.Email, req.IPAddress)
        return nil, errors.New("invalid credentials")
    }
    
    // Step 2: Check account status
    if user.Status != "active" {
        return nil, errors.New("account suspended")
    }
    
    // Step 3: Verify MFA if enabled
    if user.MFAEnabled {
        if req.MFACode == "" {
            return &AuthResponse{RequiresMFA: true}, nil
        }
        
        valid, err := as.mfaProvider.ValidateCode(user.ID, req.MFACode)
        if err != nil || !valid {
            return nil, errors.New("invalid MFA code")
        }
    }
    
    // Step 4: Generate JWT token
    token, err := as.jwtManager.GenerateToken(user.ID, user.Role)
    if err != nil {
        return nil, err
    }
    
    // Step 5: Create session
    session := &Session{
        UserID:    user.ID,
        DeviceID:  req.DeviceID,
        IPAddress: req.IPAddress,
        UserAgent: req.UserAgent,
        CreatedAt: time.Now(),
        ExpiresAt: time.Now().Add(24 * time.Hour),
    }
    
    err = as.sessionStore.CreateSession(session)
    if err != nil {
        return nil, err
    }
    
    return &AuthResponse{
        Token:        token,
        RefreshToken: session.RefreshToken,
        ExpiresAt:    session.ExpiresAt,
        User:         user,
    }, nil
}
```

### Role-Based Access Control (RBAC)
```go
type Permission struct {
    Resource string `json:"resource"` // "photos", "albums", "users"
    Action   string `json:"action"`   // "read", "write", "delete", "share"
    Scope    string `json:"scope"`    // "own", "shared", "public", "all"
}

type Role struct {
    ID          string       `json:"id"`
    Name        string       `json:"name"`
    Permissions []Permission `json:"permissions"`
}

var Roles = map[string]Role{
    "user": {
        ID:   "user",
        Name: "Regular User",
        Permissions: []Permission{
            {Resource: "photos", Action: "read", Scope: "own"},
            {Resource: "photos", Action: "write", Scope: "own"},
            {Resource: "photos", Action: "delete", Scope: "own"},
            {Resource: "photos", Action: "share", Scope: "own"},
            {Resource: "albums", Action: "read", Scope: "own"},
            {Resource: "albums", Action: "write", Scope: "own"},
        },
    },
    "admin": {
        ID:   "admin",
        Name: "Administrator",
        Permissions: []Permission{
            {Resource: "*", Action: "*", Scope: "all"},
        },
    },
}

type AuthorizationService struct {
    roleStore RoleStore
}

func (as *AuthorizationService) HasPermission(userID, resource, action string) (bool, error) {
    user, err := as.roleStore.GetUser(userID)
    if err != nil {
        return false, err
    }
    
    role, exists := Roles[user.Role]
    if !exists {
        return false, errors.New("invalid role")
    }
    
    for _, permission := range role.Permissions {
        if as.matchesPermission(permission, resource, action) {
            return true, nil
        }
    }
    
    return false, nil
}

func (as *AuthorizationService) matchesPermission(perm Permission, resource, action string) bool {
    return (perm.Resource == "*" || perm.Resource == resource) &&
           (perm.Action == "*" || perm.Action == action)
}
```

## Data Encryption

### Encryption at Rest
```go
type EncryptionService struct {
    kmsClient  KMSClient
    cipher     cipher.AEAD
    keyRotator *KeyRotator
}

type EncryptedData struct {
    Ciphertext []byte `json:"ciphertext"`
    KeyID      string `json:"key_id"`
    Nonce      []byte `json:"nonce"`
    Algorithm  string `json:"algorithm"`
}

func (es *EncryptionService) EncryptPhoto(photoData []byte, userID string) (*EncryptedData, error) {
    // Get user-specific encryption key
    keyID := es.getUserKeyID(userID)
    key, err := es.kmsClient.GetKey(keyID)
    if err != nil {
        return nil, fmt.Errorf("failed to get encryption key: %w", err)
    }
    
    // Generate random nonce
    nonce := make([]byte, es.cipher.NonceSize())
    if _, err := io.ReadFull(rand.Reader, nonce); err != nil {
        return nil, err
    }
    
    // Encrypt photo data
    ciphertext := es.cipher.Seal(nil, nonce, photoData, nil)
    
    return &EncryptedData{
        Ciphertext: ciphertext,
        KeyID:      keyID,
        Nonce:      nonce,
        Algorithm:  "AES-256-GCM",
    }, nil
}

func (es *EncryptionService) DecryptPhoto(encData *EncryptedData) ([]byte, error) {
    // Get decryption key
    key, err := es.kmsClient.GetKey(encData.KeyID)
    if err != nil {
        return nil, fmt.Errorf("failed to get decryption key: %w", err)
    }
    
    // Decrypt data
    plaintext, err := es.cipher.Open(nil, encData.Nonce, encData.Ciphertext, nil)
    if err != nil {
        return nil, fmt.Errorf("decryption failed: %w", err)
    }
    
    return plaintext, nil
}
```

### Encryption in Transit
```go
type TLSConfig struct {
    CertFile       string
    KeyFile        string
    MinVersion     uint16
    CipherSuites   []uint16
    CurvePrefs     []tls.CurveID
}

func (tc *TLSConfig) GetTLSConfig() *tls.Config {
    return &tls.Config{
        MinVersion: tls.VersionTLS13,
        CipherSuites: []uint16{
            tls.TLS_AES_256_GCM_SHA384,
            tls.TLS_AES_128_GCM_SHA256,
            tls.TLS_CHACHA20_POLY1305_SHA256,
        },
        CurvePreferences: []tls.CurveID{
            tls.X25519,
            tls.CurveP384,
            tls.CurveP256,
        },
        PreferServerCipherSuites: true,
    }
}
```

## Input Validation and Sanitization

### File Upload Security
```go
type FileValidator struct {
    maxFileSize    int64
    allowedTypes   map[string]bool
    virusScanner   VirusScanner
    imageValidator ImageValidator
}

func (fv *FileValidator) ValidateUpload(file *UploadedFile) error {
    // Check file size
    if file.Size > fv.maxFileSize {
        return errors.New("file too large")
    }
    
    // Validate MIME type
    if !fv.allowedTypes[file.MimeType] {
        return errors.New("unsupported file type")
    }
    
    // Virus scanning
    if fv.virusScanner.IsInfected(file.Content) {
        return errors.New("malicious file detected")
    }
    
    // Image-specific validation
    if strings.HasPrefix(file.MimeType, "image/") {
        if err := fv.imageValidator.ValidateImage(file.Content); err != nil {
            return fmt.Errorf("invalid image: %w", err)
        }
    }
    
    // Check for embedded scripts/malware
    if fv.containsMaliciousContent(file.Content) {
        return errors.New("malicious content detected")
    }
    
    return nil
}

func (fv *FileValidator) containsMaliciousContent(content []byte) bool {
    // Check for script tags, PHP code, etc.
    maliciousPatterns := []string{
        "<script",
        "<?php",
        "<%",
        "javascript:",
        "vbscript:",
    }
    
    contentStr := strings.ToLower(string(content))
    for _, pattern := range maliciousPatterns {
        if strings.Contains(contentStr, pattern) {
            return true
        }
    }
    
    return false
}
```

### SQL Injection Prevention
```go
type SafeQuery struct {
    db *sql.DB
}

func (sq *SafeQuery) GetUserPhotos(userID string, limit, offset int) ([]*Photo, error) {
    // Use parameterized queries to prevent SQL injection
    query := `
        SELECT id, filename, size, uploaded_at, metadata
        FROM photos 
        WHERE user_id = $1 
        ORDER BY uploaded_at DESC 
        LIMIT $2 OFFSET $3
    `
    
    rows, err := sq.db.Query(query, userID, limit, offset)
    if err != nil {
        return nil, err
    }
    defer rows.Close()
    
    var photos []*Photo
    for rows.Next() {
        photo := &Photo{}
        err := rows.Scan(&photo.ID, &photo.Filename, &photo.Size, 
                        &photo.UploadedAt, &photo.Metadata)
        if err != nil {
            return nil, err
        }
        photos = append(photos, photo)
    }
    
    return photos, nil
}
```

## Rate Limiting and DDoS Protection

### Rate Limiting Implementation
```go
type RateLimiter struct {
    store    RateLimitStore
    rules    []RateLimitRule
    redis    RedisClient
}

type RateLimitRule struct {
    Endpoint    string        `json:"endpoint"`
    Method      string        `json:"method"`
    Limit       int           `json:"limit"`
    Window      time.Duration `json:"window"`
    Scope       string        `json:"scope"` // "ip", "user", "api_key"
}

func (rl *RateLimiter) CheckLimit(request *http.Request) (*RateLimitResult, error) {
    rule := rl.findMatchingRule(request)
    if rule == nil {
        return &RateLimitResult{Allowed: true}, nil
    }
    
    key := rl.generateKey(request, rule)
    
    // Use sliding window log algorithm
    now := time.Now()
    windowStart := now.Add(-rule.Window)
    
    // Remove old entries
    rl.redis.ZRemRangeByScore(key, 0, float64(windowStart.Unix()))
    
    // Count current requests
    count, err := rl.redis.ZCard(key)
    if err != nil {
        return nil, err
    }
    
    if count >= int64(rule.Limit) {
        return &RateLimitResult{
            Allowed:   false,
            Remaining: 0,
            ResetTime: windowStart.Add(rule.Window),
        }, nil
    }
    
    // Add current request
    rl.redis.ZAdd(key, &redis.Z{
        Score:  float64(now.Unix()),
        Member: now.UnixNano(),
    })
    
    // Set expiration
    rl.redis.Expire(key, rule.Window)
    
    return &RateLimitResult{
        Allowed:   true,
        Remaining: rule.Limit - int(count) - 1,
        ResetTime: windowStart.Add(rule.Window),
    }, nil
}
```

### DDoS Protection
```go
type DDoSProtection struct {
    ipTracker    *IPTracker
    geoLocation  GeoLocationService
    blacklist    *IPBlacklist
    cloudflare   CloudflareAPI
}

func (ddos *DDoSProtection) AnalyzeRequest(request *http.Request) (*ThreatAnalysis, error) {
    clientIP := extractClientIP(request)
    
    analysis := &ThreatAnalysis{
        IP:        clientIP,
        Timestamp: time.Now(),
    }
    
    // Check IP reputation
    if ddos.blacklist.IsBlacklisted(clientIP) {
        analysis.ThreatLevel = "HIGH"
        analysis.Reasons = append(analysis.Reasons, "blacklisted_ip")
        return analysis, nil
    }
    
    // Analyze request patterns
    pattern := ddos.ipTracker.GetRequestPattern(clientIP)
    if pattern.RequestsPerSecond > 100 {
        analysis.ThreatLevel = "HIGH"
        analysis.Reasons = append(analysis.Reasons, "high_request_rate")
    }
    
    // Check geographic anomalies
    location := ddos.geoLocation.GetLocation(clientIP)
    if ddos.isAnomalousLocation(clientIP, location) {
        analysis.ThreatLevel = "MEDIUM"
        analysis.Reasons = append(analysis.Reasons, "geographic_anomaly")
    }
    
    // Bot detection
    if ddos.isBotTraffic(request) {
        analysis.ThreatLevel = "MEDIUM"
        analysis.Reasons = append(analysis.Reasons, "bot_traffic")
    }
    
    return analysis, nil
}

func (ddos *DDoSProtection) MitigateAttack(analysis *ThreatAnalysis) error {
    switch analysis.ThreatLevel {
    case "HIGH":
        // Block IP at edge
        return ddos.cloudflare.BlockIP(analysis.IP, 24*time.Hour)
    case "MEDIUM":
        // Apply rate limiting
        return ddos.cloudflare.ApplyRateLimit(analysis.IP, 10)
    default:
        return nil
    }
}
```

## Privacy and Compliance

### GDPR Compliance
```go
type GDPRService struct {
    dataProcessor DataProcessor
    consentManager ConsentManager
    auditLogger   AuditLogger
}

type DataSubjectRequest struct {
    Type      string    `json:"type"` // "access", "rectification", "erasure", "portability"
    UserID    string    `json:"user_id"`
    RequestID string    `json:"request_id"`
    Timestamp time.Time `json:"timestamp"`
}

func (gdpr *GDPRService) ProcessDataSubjectRequest(request *DataSubjectRequest) error {
    switch request.Type {
    case "access":
        return gdpr.handleDataAccess(request)
    case "rectification":
        return gdpr.handleDataRectification(request)
    case "erasure":
        return gdpr.handleDataErasure(request)
    case "portability":
        return gdpr.handleDataPortability(request)
    default:
        return errors.New("invalid request type")
    }
}

func (gdpr *GDPRService) handleDataErasure(request *DataSubjectRequest) error {
    // Log the request
    gdpr.auditLogger.LogGDPRRequest(request)
    
    // Get all user data
    userData, err := gdpr.dataProcessor.GetAllUserData(request.UserID)
    if err != nil {
        return err
    }
    
    // Delete photos from object storage
    for _, photo := range userData.Photos {
        err := gdpr.dataProcessor.DeletePhotoFromStorage(photo.StoragePath)
        if err != nil {
            return err
        }
    }
    
    // Delete metadata from database
    err = gdpr.dataProcessor.DeleteUserMetadata(request.UserID)
    if err != nil {
        return err
    }
    
    // Delete from search index
    err = gdpr.dataProcessor.DeleteFromSearchIndex(request.UserID)
    if err != nil {
        return err
    }
    
    // Delete from cache
    err = gdpr.dataProcessor.DeleteFromCache(request.UserID)
    if err != nil {
        return err
    }
    
    // Log completion
    gdpr.auditLogger.LogGDPRCompletion(request)
    
    return nil
}
```

### Data Anonymization
```go
type AnonymizationService struct {
    crypto CryptoService
}

func (as *AnonymizationService) AnonymizeMetadata(metadata *PhotoMetadata) *PhotoMetadata {
    anonymized := *metadata
    
    // Remove or hash personally identifiable information
    if metadata.Location != nil {
        // Reduce GPS precision to city level
        anonymized.Location = &LocationInfo{
            City:    metadata.Location.City,
            Country: metadata.Location.Country,
            // Remove exact coordinates
        }
    }
    
    // Remove camera serial numbers
    if anonymized.Camera != nil {
        anonymized.Camera.SerialNumber = ""
    }
    
    // Hash any remaining identifiers
    if anonymized.DeviceID != "" {
        anonymized.DeviceID = as.crypto.Hash(anonymized.DeviceID)
    }
    
    return &anonymized
}
```

## Audit Logging

### Comprehensive Audit Trail
```go
type AuditLogger struct {
    store   AuditStore
    encoder LogEncoder
}

type AuditEvent struct {
    ID          string                 `json:"id"`
    Timestamp   time.Time              `json:"timestamp"`
    UserID      string                 `json:"user_id"`
    Action      string                 `json:"action"`
    Resource    string                 `json:"resource"`
    IPAddress   string                 `json:"ip_address"`
    UserAgent   string                 `json:"user_agent"`
    Success     bool                   `json:"success"`
    Details     map[string]interface{} `json:"details"`
    Fingerprint string                 `json:"fingerprint"`
}

func (al *AuditLogger) LogUserAction(userID, action, resource string, 
    request *http.Request, success bool, details map[string]interface{}) {
    
    event := &AuditEvent{
        ID:        generateUUID(),
        Timestamp: time.Now(),
        UserID:    userID,
        Action:    action,
        Resource:  resource,
        IPAddress: extractClientIP(request),
        UserAgent: request.UserAgent(),
        Success:   success,
        Details:   details,
        Fingerprint: al.generateFingerprint(userID, action, resource),
    }
    
    // Store in secure audit store
    err := al.store.Store(event)
    if err != nil {
        log.Error("Failed to store audit event", "error", err, "event", event)
    }
    
    // Also send to SIEM system for real-time monitoring
    al.sendToSIEM(event)
}

func (al *AuditLogger) generateFingerprint(userID, action, resource string) string {
    data := fmt.Sprintf("%s:%s:%s:%d", userID, action, resource, time.Now().Unix())
    hash := sha256.Sum256([]byte(data))
    return hex.EncodeToString(hash[:])
}
```

## Security Monitoring

### Intrusion Detection
```go
type SecurityMonitor struct {
    alertManager AlertManager
    riskScorer   RiskScorer
    mlDetector   MLAnomalyDetector
}

type SecurityAlert struct {
    ID          string    `json:"id"`
    Severity    string    `json:"severity"` // "LOW", "MEDIUM", "HIGH", "CRITICAL"
    Type        string    `json:"type"`
    Description string    `json:"description"`
    UserID      string    `json:"user_id"`
    IPAddress   string    `json:"ip_address"`
    Timestamp   time.Time `json:"timestamp"`
    Evidence    []Evidence `json:"evidence"`
}

func (sm *SecurityMonitor) DetectAnomalies(events []AuditEvent) []*SecurityAlert {
    var alerts []*SecurityAlert
    
    for _, event := range events {
        // Check for known attack patterns
        if sm.isKnownAttackPattern(event) {
            alert := &SecurityAlert{
                ID:          generateUUID(),
                Severity:    "HIGH",
                Type:        "KNOWN_ATTACK_PATTERN",
                Description: fmt.Sprintf("Detected known attack pattern: %s", event.Action),
                UserID:      event.UserID,
                IPAddress:   event.IPAddress,
                Timestamp:   event.Timestamp,
            }
            alerts = append(alerts, alert)
        }
        
        // Use ML for anomaly detection
        anomalyScore := sm.mlDetector.CalculateAnomalyScore(event)
        if anomalyScore > 0.8 {
            alert := &SecurityAlert{
                ID:          generateUUID(),
                Severity:    sm.mapScoreToSeverity(anomalyScore),
                Type:        "BEHAVIORAL_ANOMALY",
                Description: fmt.Sprintf("Unusual user behavior detected (score: %.2f)", anomalyScore),
                UserID:      event.UserID,
                IPAddress:   event.IPAddress,
                Timestamp:   event.Timestamp,
            }
            alerts = append(alerts, alert)
        }
    }
    
    return alerts
}

func (sm *SecurityMonitor) isKnownAttackPattern(event AuditEvent) bool {
    // SQL injection patterns
    if strings.Contains(strings.ToLower(event.Resource), "union select") ||
       strings.Contains(strings.ToLower(event.Resource), "drop table") {
        return true
    }
    
    // XSS patterns
    if strings.Contains(strings.ToLower(event.Resource), "<script") ||
       strings.Contains(strings.ToLower(event.Resource), "javascript:") {
        return true
    }
    
    // Path traversal
    if strings.Contains(event.Resource, "../") ||
       strings.Contains(event.Resource, "..\\") {
        return true
    }
    
    return false
}
```

## Incident Response

### Automated Response System
```go
type IncidentResponseSystem struct {
    playbooks    map[string]ResponsePlaybook
    notifier     AlertNotifier
    blocker      TrafficBlocker
    quarantine   QuarantineService
}

type ResponsePlaybook struct {
    Name    string           `json:"name"`
    Trigger TriggerCondition `json:"trigger"`
    Actions []ResponseAction `json:"actions"`
}

func (irs *IncidentResponseSystem) HandleAlert(alert *SecurityAlert) error {
    playbook := irs.findPlaybook(alert)
    if playbook == nil {
        return errors.New("no playbook found for alert type")
    }
    
    for _, action := range playbook.Actions {
        err := irs.executeAction(action, alert)
        if err != nil {
            log.Error("Failed to execute response action", 
                     "action", action.Type, "error", err)
            continue
        }
    }
    
    return nil
}

func (irs *IncidentResponseSystem) executeAction(action ResponseAction, alert *SecurityAlert) error {
    switch action.Type {
    case "BLOCK_IP":
        return irs.blocker.BlockIP(alert.IPAddress, action.Duration)
    case "QUARANTINE_USER":
        return irs.quarantine.QuarantineUser(alert.UserID, action.Reason)
    case "NOTIFY_ADMIN":
        return irs.notifier.NotifyAdmins(alert)
    case "ESCALATE":
        return irs.notifier.EscalateToSecurity(alert)
    default:
        return fmt.Errorf("unknown action type: %s", action.Type)
    }
}
```

This comprehensive security framework provides multiple layers of protection while ensuring compliance with privacy regulations and maintaining detailed audit trails for forensic analysis.
