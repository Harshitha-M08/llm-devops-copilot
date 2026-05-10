# Security Documentation - AI-Driven Hybrid Kubernetes System

## Table of Contents
1. [Security Overview](#security-overview)
2. [Security Best Practices](#security-best-practices)
3. [Secret Management](#secret-management)
4. [Network Security](#network-security)
5. [Access Control](#access-control)
6. [Compliance Considerations](#compliance-considerations)
7. [Vulnerability Management](#vulnerability-management)
8. [Security Audit Checklist](#security-audit-checklist)
9. [Incident Response](#incident-response)

---

## Security Overview

The AI-Driven Hybrid Kubernetes System implements a defense-in-depth security strategy with multiple layers of protection:

```
┌─────────────────────────────────────────────────────────┐
│ Layer 7: Monitoring & Audit                            │
│  • Security logging                                     │
│  • Audit trails                                         │
│  • Anomaly detection                                    │
└─────────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────────┐
│ Layer 6: Application Security                          │
│  • Input validation                                     │
│  • Output encoding                                      │
│  • Secure coding practices                              │
└─────────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────────┐
│ Layer 5: API Security                                   │
│  • Authentication (JWT)                                 │
│  • Authorization (RBAC)                                 │
│  • Rate limiting                                        │
└─────────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────────┐
│ Layer 4: Data Security                                  │
│  • Encryption at rest                                   │
│  • Encryption in transit                                │
│  • Secret management                                    │
└─────────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────────┐
│ Layer 3: Container Security                             │
│  • Image scanning                                       │
│  • Security contexts                                    │
│  • Pod security policies                                │
└─────────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────────┐
│ Layer 2: Network Security                               │
│  • Network policies                                     │
│  • TLS/SSL                                              │
│  • Firewall rules                                       │
└─────────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────────┐
│ Layer 1: Infrastructure Security                        │
│  • RBAC for Kubernetes                                  │
│  • Node security                                        │
│  • Cloud provider security                              │
└─────────────────────────────────────────────────────────┘
```

### Security Principles

1. **Least Privilege**: Grant minimum necessary permissions
2. **Defense in Depth**: Multiple security layers
3. **Secure by Default**: Secure configuration from the start
4. **Zero Trust**: Verify every request
5. **Regular Updates**: Keep systems patched and updated

---

## Security Best Practices

### 1. Container Security

#### Image Security

**Use Official Base Images**:
```dockerfile
# Good: Official, minimal base image
FROM python:3.11-alpine

# Bad: Latest tag, unknown source
FROM someuser/python:latest
```

**Scan Images for Vulnerabilities**:
```bash
# Using Trivy
trivy image your-registry/llm-service:v1.0.0

# Using Grype
grype your-registry/llm-service:v1.0.0

# Using Snyk
snyk container test your-registry/llm-service:v1.0.0
```

**Sign and Verify Images**:
```bash
# Sign with Cosign
cosign sign --key cosign.key your-registry/llm-service:v1.0.0

# Verify signature
cosign verify --key cosign.pub your-registry/llm-service:v1.0.0
```

**Minimal Container Images**:
```dockerfile
# Multi-stage build for minimal final image
FROM python:3.11 AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --user --no-cache-dir -r requirements.txt

FROM python:3.11-alpine
WORKDIR /app
COPY --from=builder /root/.local /root/.local
COPY . .

# Run as non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

# Make filesystem read-only
ENV PATH=/root/.local/bin:$PATH
EXPOSE 8000
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

#### Security Context

**Pod Security Context**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: llm-service
spec:
  template:
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        runAsGroup: 1000
        fsGroup: 1000
        seccompProfile:
          type: RuntimeDefault
      containers:
      - name: llm-service
        image: llm-service:v1.0.0
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          runAsNonRoot: true
          capabilities:
            drop:
              - ALL
          seccompProfile:
            type: RuntimeDefault
        volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: cache
          mountPath: /app/cache
      volumes:
      - name: tmp
        emptyDir: {}
      - name: cache
        emptyDir: {}
```

#### Pod Security Standards

**Enforce Pod Security Standards**:
```yaml
# namespace-security.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: ai-system
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

**Pod Security Policy (deprecated in k8s 1.25, use Pod Security Admission)**:
```yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: restricted
spec:
  privileged: false
  allowPrivilegeEscalation: false
  requiredDropCapabilities:
    - ALL
  volumes:
    - 'configMap'
    - 'emptyDir'
    - 'projected'
    - 'secret'
    - 'downwardAPI'
    - 'persistentVolumeClaim'
  runAsUser:
    rule: 'MustRunAsNonRoot'
  seLinux:
    rule: 'RunAsAny'
  fsGroup:
    rule: 'RunAsAny'
  readOnlyRootFilesystem: true
```

### 2. API Security

#### Authentication

**JWT Configuration**:
```javascript
// Strong JWT configuration
const jwt = require('jsonwebtoken');

// Generate token
const generateToken = (user) => {
  return jwt.sign(
    {
      userId: user.id,
      username: user.username,
      role: user.role,
      iat: Math.floor(Date.now() / 1000),
    },
    process.env.JWT_SECRET,
    {
      algorithm: 'RS256',  // Use asymmetric algorithm
      expiresIn: '15m',    // Short-lived tokens
      issuer: 'ai-system',
      audience: 'ai-system-api',
    }
  );
};

// Verify token
const verifyToken = (token) => {
  return jwt.verify(token, process.env.JWT_PUBLIC_KEY, {
    algorithms: ['RS256'],
    issuer: 'ai-system',
    audience: 'ai-system-api',
  });
};
```

**Password Security**:
```javascript
const bcrypt = require('bcrypt');

// Hash password
const hashPassword = async (password) => {
  const saltRounds = 12;  // Increase for more security
  return await bcrypt.hash(password, saltRounds);
};

// Verify password
const verifyPassword = async (password, hash) => {
  return await bcrypt.compare(password, hash);
};

// Password strength validation
const validatePassword = (password) => {
  const minLength = 12;
  const hasUpperCase = /[A-Z]/.test(password);
  const hasLowerCase = /[a-z]/.test(password);
  const hasNumbers = /\d/.test(password);
  const hasSpecialChar = /[!@#$%^&*(),.?":{}|<>]/.test(password);

  return (
    password.length >= minLength &&
    hasUpperCase &&
    hasLowerCase &&
    hasNumbers &&
    hasSpecialChar
  );
};
```

#### Input Validation

**Python (Pydantic)**:
```python
from pydantic import BaseModel, Field, validator
import re

class UserCreate(BaseModel):
    username: str = Field(..., min_length=3, max_length=50)
    email: str = Field(..., regex=r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$')
    password: str = Field(..., min_length=12)

    @validator('username')
    def username_alphanumeric(cls, v):
        if not re.match(r'^[a-zA-Z0-9_-]+$', v):
            raise ValueError('Username must be alphanumeric')
        return v

    @validator('password')
    def password_strength(cls, v):
        if not re.search(r'[A-Z]', v):
            raise ValueError('Password must contain uppercase letter')
        if not re.search(r'[a-z]', v):
            raise ValueError('Password must contain lowercase letter')
        if not re.search(r'\d', v):
            raise ValueError('Password must contain digit')
        if not re.search(r'[!@#$%^&*(),.?":{}|<>]', v):
            raise ValueError('Password must contain special character')
        return v
```

**SQL Injection Prevention**:
```python
# Good: Parameterized query
cursor.execute(
    "SELECT * FROM users WHERE username = %s AND email = %s",
    (username, email)
)

# Bad: String concatenation
# cursor.execute(f"SELECT * FROM users WHERE username = '{username}'")
```

**XSS Prevention**:
```javascript
// Sanitize user input
const sanitizeHtml = require('sanitize-html');

const sanitize = (input) => {
  return sanitizeHtml(input, {
    allowedTags: [],
    allowedAttributes: {},
  });
};

// Set security headers
app.use((req, res, next) => {
  res.setHeader('X-Content-Type-Options', 'nosniff');
  res.setHeader('X-Frame-Options', 'DENY');
  res.setHeader('X-XSS-Protection', '1; mode=block');
  res.setHeader('Content-Security-Policy', "default-src 'self'");
  next();
});
```

#### Rate Limiting

**Application-Level Rate Limiting**:
```javascript
const rateLimit = require('express-rate-limit');
const RedisStore = require('rate-limit-redis');

// Global rate limiter
const limiter = rateLimit({
  store: new RedisStore({
    client: redisClient,
    prefix: 'rate-limit:',
  }),
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100, // Limit each IP to 100 requests per windowMs
  message: 'Too many requests from this IP',
  standardHeaders: true,
  legacyHeaders: false,
});

// Endpoint-specific rate limiter
const apiLimiter = rateLimit({
  store: new RedisStore({ client: redisClient }),
  windowMs: 60 * 1000, // 1 minute
  max: 10, // 10 requests per minute
  skipSuccessfulRequests: false,
});

app.use('/api/', limiter);
app.use('/api/auth/login', apiLimiter);
```

**Ingress Rate Limiting**:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ai-system-ingress
  annotations:
    nginx.ingress.kubernetes.io/limit-rps: "100"
    nginx.ingress.kubernetes.io/limit-connections: "10"
    nginx.ingress.kubernetes.io/limit-burst-multiplier: "5"
```

---

## Secret Management

### Kubernetes Secrets

**Creating Secrets Securely**:
```bash
# From literal values
kubectl create secret generic api-keys \
  --from-literal=openai-key='sk-...' \
  --from-literal=anthropic-key='sk-ant-...' \
  -n ai-system

# From file
kubectl create secret generic db-credentials \
  --from-file=username=./db-user.txt \
  --from-file=password=./db-pass.txt \
  -n ai-system

# From env file
kubectl create secret generic app-secrets \
  --from-env-file=.env \
  -n ai-system
```

**Enable Encryption at Rest**:
```yaml
# encryption-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: <base64-encoded-32-byte-key>
      - identity: {}
```

```bash
# Apply encryption config (managed differently per platform)
# For kubeadm clusters:
# Add to /etc/kubernetes/manifests/kube-apiserver.yaml
--encryption-provider-config=/etc/kubernetes/encryption-config.yaml
```

### Sealed Secrets

**Install Sealed Secrets Controller**:
```bash
# Install controller
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm install sealed-secrets sealed-secrets/sealed-secrets \
  --namespace kube-system

# Install kubeseal CLI
wget https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.24.0/kubeseal-0.24.0-linux-amd64.tar.gz
tar -xvzf kubeseal-0.24.0-linux-amd64.tar.gz
sudo mv kubeseal /usr/local/bin/
```

**Create Sealed Secret**:
```bash
# Create regular secret (don't apply)
kubectl create secret generic api-keys \
  --from-literal=openai-key='sk-...' \
  --dry-run=client \
  -o yaml > secret.yaml

# Seal the secret
kubeseal -f secret.yaml -o yaml > sealed-secret.yaml

# Safe to commit sealed-secret.yaml to Git
git add sealed-secret.yaml

# Apply to cluster
kubectl apply -f sealed-secret.yaml -n ai-system
```

### External Secrets Operator

**Install External Secrets Operator**:
```bash
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets \
  --namespace external-secrets-system \
  --create-namespace
```

**AWS Secrets Manager Integration**:
```yaml
# secret-store.yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-secrets-manager
  namespace: ai-system
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets-sa
---
# external-secret.yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: ai-system-secrets
  namespace: ai-system
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: SecretStore
  target:
    name: ai-system-secrets
    creationPolicy: Owner
  data:
  - secretKey: POSTGRES_PASSWORD
    remoteRef:
      key: ai-system/postgres-password
  - secretKey: OPENAI_API_KEY
    remoteRef:
      key: ai-system/openai-api-key
  - secretKey: JWT_SECRET
    remoteRef:
      key: ai-system/jwt-secret
```

**HashiCorp Vault Integration**:
```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: vault-backend
  namespace: ai-system
spec:
  provider:
    vault:
      server: "https://vault.example.com"
      path: "secret"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "ai-system-role"
          serviceAccountRef:
            name: external-secrets-sa
```

### Secret Rotation

**Automated Secret Rotation**:
```yaml
# secret-rotation-cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: rotate-secrets
  namespace: ai-system
spec:
  schedule: "0 0 1 * *"  # Monthly on 1st at midnight
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: secret-rotator
          containers:
          - name: rotator
            image: secret-rotator:v1.0.0
            env:
            - name: SECRETS_TO_ROTATE
              value: "JWT_SECRET,API_KEY,ENCRYPTION_KEY"
            - name: SECRET_MANAGER
              value: "aws-secrets-manager"
          restartPolicy: OnFailure
```

### Secret Best Practices

1. **Never commit secrets to Git**
   ```bash
   # Add to .gitignore
   *.env
   secrets.yaml
   .secrets/
   *.key
   *.pem
   ```

2. **Use environment-specific secrets**
   - Development: Local or test credentials
   - Staging: Staging-specific credentials
   - Production: Production credentials with restricted access

3. **Rotate secrets regularly**
   - API keys: Every 90 days
   - Passwords: Every 90 days
   - Certificates: Before expiration

4. **Limit secret access**
   ```yaml
   # Only grant secret access to pods that need it
   apiVersion: v1
   kind: ServiceAccount
   metadata:
     name: llm-service-sa
   ---
   apiVersion: rbac.authorization.k8s.io/v1
   kind: Role
   metadata:
     name: secret-reader
   rules:
   - apiGroups: [""]
     resources: ["secrets"]
     resourceNames: ["ai-system-secrets"]
     verbs: ["get"]
   ---
   apiVersion: rbac.authorization.k8s.io/v1
   kind: RoleBinding
   metadata:
     name: llm-service-secret-reader
   subjects:
   - kind: ServiceAccount
     name: llm-service-sa
   roleRef:
     kind: Role
     name: secret-reader
     apiGroup: rbac.authorization.k8s.io
   ```

---

## Network Security

### Network Policies

**Default Deny All**:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: ai-system
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

**Allow Specific Traffic**:
```yaml
# Allow frontend to backend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: ai-system
spec:
  podSelector:
    matchLabels:
      app: approval-backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: approval-frontend
    ports:
    - protocol: TCP
      port: 3000

---
# Allow egress to external APIs
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-llm-service-egress
  namespace: ai-system
spec:
  podSelector:
    matchLabels:
      app: llm-service
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector: {}
    ports:
    - protocol: TCP
      port: 443  # HTTPS
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53  # DNS
```

### TLS/SSL Configuration

**Generate TLS Certificates**:
```bash
# Using cert-manager with Let's Encrypt
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml

# Create ClusterIssuer
cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@example.com
    privateKeySecretRef:
      name: letsencrypt-prod-key
    solvers:
    - http01:
        ingress:
          class: nginx
EOF

# Create Certificate
cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: ai-system-tls
  namespace: ai-system
spec:
  secretName: ai-system-tls
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
  - ai-system.example.com
  - api.ai-system.example.com
EOF
```

**Ingress TLS Configuration**:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ai-system-ingress
  namespace: ai-system
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    nginx.ingress.kubernetes.io/ssl-protocols: "TLSv1.3"
    nginx.ingress.kubernetes.io/ssl-ciphers: "ECDHE-RSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - ai-system.example.com
    - api.ai-system.example.com
    secretName: ai-system-tls
  rules:
  - host: ai-system.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: approval-frontend
            port:
              number: 80
```

**Service Mesh mTLS (with Istio)**:
```yaml
# Enable mTLS for all services in namespace
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: ai-system
spec:
  mtls:
    mode: STRICT
```

### Firewall Rules

**AWS Security Groups**:
```bash
# Allow HTTPS from anywhere
aws ec2 authorize-security-group-ingress \
  --group-id sg-xxxxx \
  --protocol tcp \
  --port 443 \
  --cidr 0.0.0.0/0

# Allow SSH from specific IP
aws ec2 authorize-security-group-ingress \
  --group-id sg-xxxxx \
  --protocol tcp \
  --port 22 \
  --cidr 1.2.3.4/32

# Deny all other inbound traffic (default)
```

**GCP Firewall Rules**:
```bash
# Create firewall rule for HTTPS
gcloud compute firewall-rules create allow-https \
  --allow tcp:443 \
  --source-ranges 0.0.0.0/0 \
  --target-tags ai-system-cluster

# Create rule for internal communication
gcloud compute firewall-rules create allow-internal \
  --allow tcp,udp,icmp \
  --source-ranges 10.0.0.0/8 \
  --target-tags ai-system-cluster
```

---

## Access Control

### Kubernetes RBAC

**Service Account for Services**:
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: llm-service-sa
  namespace: ai-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: llm-service-role
  namespace: ai-system
rules:
- apiGroups: [""]
  resources: ["secrets", "configmaps"]
  verbs: ["get", "list"]
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: llm-service-rolebinding
  namespace: ai-system
subjects:
- kind: ServiceAccount
  name: llm-service-sa
roleRef:
  kind: Role
  name: llm-service-role
  apiGroup: rbac.authorization.k8s.io
```

**Cluster-wide Admin Role** (use sparingly):
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-admin
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user-binding
subjects:
- kind: User
  name: admin@example.com
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
```

**Read-Only Role**:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: viewer
  namespace: ai-system
rules:
- apiGroups: ["", "apps", "batch"]
  resources: ["*"]
  verbs: ["get", "list", "watch"]
```

### Application-Level RBAC

**Role Definitions**:
```javascript
const ROLES = {
  ADMIN: 'admin',
  APPROVER: 'approver',
  VIEWER: 'viewer',
  SERVICE: 'service',
};

const PERMISSIONS = {
  APPROVE_REQUEST: 'approve_request',
  DENY_REQUEST: 'deny_request',
  VIEW_APPROVALS: 'view_approvals',
  CREATE_USER: 'create_user',
  DELETE_USER: 'delete_user',
  VIEW_AUDIT_LOGS: 'view_audit_logs',
};

const rolePermissions = {
  [ROLES.ADMIN]: Object.values(PERMISSIONS),
  [ROLES.APPROVER]: [
    PERMISSIONS.APPROVE_REQUEST,
    PERMISSIONS.DENY_REQUEST,
    PERMISSIONS.VIEW_APPROVALS,
    PERMISSIONS.VIEW_AUDIT_LOGS,
  ],
  [ROLES.VIEWER]: [PERMISSIONS.VIEW_APPROVALS],
  [ROLES.SERVICE]: [PERMISSIONS.VIEW_APPROVALS],
};
```

**Authorization Middleware**:
```javascript
const requirePermission = (permission) => {
  return (req, res, next) => {
    const userRole = req.user.role;
    const permissions = rolePermissions[userRole] || [];

    if (!permissions.includes(permission)) {
      return res.status(403).json({
        error: {
          code: 'AUTHORIZATION_ERROR',
          message: 'Insufficient permissions',
        },
      });
    }

    next();
  };
};

// Usage
app.post('/api/approvals/:id/approve',
  authenticate,
  requirePermission(PERMISSIONS.APPROVE_REQUEST),
  approveHandler
);
```

### Multi-Factor Authentication (MFA)

**Implement TOTP-based MFA**:
```javascript
const speakeasy = require('speakeasy');
const QRCode = require('qrcode');

// Generate MFA secret for user
const generateMFASecret = (username) => {
  const secret = speakeasy.generateSecret({
    name: `AI System (${username})`,
    length: 32,
  });

  return {
    secret: secret.base32,
    otpauthUrl: secret.otpauth_url,
  };
};

// Generate QR code
const generateQRCode = async (otpauthUrl) => {
  return await QRCode.toDataURL(otpauthUrl);
};

// Verify TOTP token
const verifyMFAToken = (secret, token) => {
  return speakeasy.totp.verify({
    secret,
    encoding: 'base32',
    token,
    window: 2, // Allow 2 time steps before/after
  });
};

// Login with MFA
app.post('/api/auth/login-mfa', async (req, res) => {
  const { username, password, mfaToken } = req.body;

  // Verify password
  const user = await authenticateUser(username, password);
  if (!user) {
    return res.status(401).json({ error: 'Invalid credentials' });
  }

  // Verify MFA token
  if (!verifyMFAToken(user.mfa_secret, mfaToken)) {
    return res.status(401).json({ error: 'Invalid MFA token' });
  }

  // Generate session
  const token = generateToken(user);
  res.json({ token });
});
```

---

## Compliance Considerations

### GDPR Compliance

**Data Privacy**:
```javascript
// Implement data access request
app.get('/api/users/:userId/data', authenticate, async (req, res) => {
  if (req.user.id !== req.params.userId && req.user.role !== 'admin') {
    return res.status(403).json({ error: 'Forbidden' });
  }

  const userData = await getUserData(req.params.userId);
  res.json(userData);
});

// Implement data deletion (right to be forgotten)
app.delete('/api/users/:userId/data', authenticate, async (req, res) => {
  if (req.user.id !== req.params.userId && req.user.role !== 'admin') {
    return res.status(403).json({ error: 'Forbidden' });
  }

  await anonymizeUserData(req.params.userId);
  res.json({ message: 'Data deleted successfully' });
});

// Data retention policy
const deleteOldData = async () => {
  const retentionPeriod = 365 * 2; // 2 years
  await deleteDataOlderThan(retentionPeriod);
};
```

**Consent Management**:
```javascript
// Track user consent
const consentSchema = new Schema({
  userId: { type: String, required: true },
  consentType: { type: String, required: true },
  granted: { type: Boolean, required: true },
  timestamp: { type: Date, default: Date.now },
  ipAddress: String,
});

// Check consent before processing
const checkConsent = async (userId, consentType) => {
  const consent = await Consent.findOne({ userId, consentType });
  return consent && consent.granted;
};
```

### SOC 2 Compliance

**Audit Logging**:
```javascript
// Log all security-relevant events
const auditLog = async (event) => {
  await AuditLog.create({
    userId: event.userId,
    action: event.action,
    resourceType: event.resourceType,
    resourceId: event.resourceId,
    ipAddress: event.ipAddress,
    userAgent: event.userAgent,
    timestamp: new Date(),
    details: event.details,
  });
};

// Middleware to log all requests
app.use((req, res, next) => {
  res.on('finish', () => {
    if (req.user) {
      auditLog({
        userId: req.user.id,
        action: `${req.method} ${req.path}`,
        ipAddress: req.ip,
        userAgent: req.get('User-Agent'),
        details: {
          statusCode: res.statusCode,
          body: req.body,
        },
      });
    }
  });
  next();
});
```

**Access Reviews**:
```sql
-- Quarterly access review
SELECT
  u.username,
  u.role,
  u.created_at,
  MAX(al.created_at) as last_activity
FROM users u
LEFT JOIN audit_logs al ON u.id = al.user_id
GROUP BY u.id
ORDER BY last_activity DESC;

-- Identify inactive users
SELECT
  username,
  role,
  created_at
FROM users
WHERE id NOT IN (
  SELECT DISTINCT user_id
  FROM audit_logs
  WHERE created_at > NOW() - INTERVAL '90 days'
);
```

### HIPAA Compliance (if handling health data)

**Encryption**:
```javascript
const crypto = require('crypto');

// Encrypt sensitive data
const encrypt = (text, key) => {
  const iv = crypto.randomBytes(16);
  const cipher = crypto.createCipheriv('aes-256-gcm', Buffer.from(key, 'hex'), iv);
  let encrypted = cipher.update(text, 'utf8', 'hex');
  encrypted += cipher.final('hex');
  const authTag = cipher.getAuthTag();

  return {
    iv: iv.toString('hex'),
    encryptedData: encrypted,
    authTag: authTag.toString('hex'),
  };
};

// Decrypt sensitive data
const decrypt = (encrypted, key) => {
  const decipher = crypto.createDecipheriv(
    'aes-256-gcm',
    Buffer.from(key, 'hex'),
    Buffer.from(encrypted.iv, 'hex')
  );
  decipher.setAuthTag(Buffer.from(encrypted.authTag, 'hex'));

  let decrypted = decipher.update(encrypted.encryptedData, 'hex', 'utf8');
  decrypted += decipher.final('utf8');
  return decrypted;
};
```

---

## Vulnerability Management

### Container Image Scanning

**Integrate Scanning in CI/CD**:
```yaml
# .github/workflows/security-scan.yml
name: Security Scan

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Build image
        run: docker build -t llm-service:${{ github.sha }} services/llm-service

      - name: Run Trivy scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: llm-service:${{ github.sha }}
          format: 'sarif'
          output: 'trivy-results.sarif'
          severity: 'CRITICAL,HIGH'

      - name: Upload Trivy results to GitHub Security
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: 'trivy-results.sarif'

      - name: Fail on critical vulnerabilities
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: llm-service:${{ github.sha }}
          exit-code: '1'
          severity: 'CRITICAL'
```

**Runtime Scanning with Falco**:
```yaml
# Install Falco
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm install falco falcosecurity/falco \
  --namespace falco \
  --create-namespace

# Custom Falco rules
# falco-rules.yaml
- rule: Unauthorized Process in Container
  desc: Detect unauthorized process execution
  condition: >
    spawned_process and
    container and
    not proc.name in (python, node, uvicorn, npm)
  output: >
    Unauthorized process in container
    (user=%user.name command=%proc.cmdline container=%container.name)
  priority: WARNING
```

### Dependency Scanning

**Python Dependencies**:
```bash
# Install safety
pip install safety

# Scan for vulnerabilities
safety check

# In CI/CD
safety check --json > safety-report.json
```

**Node.js Dependencies**:
```bash
# Use npm audit
npm audit

# Fix vulnerabilities automatically
npm audit fix

# In CI/CD
npm audit --audit-level=high
```

**Automated Dependency Updates**:
```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "pip"
    directory: "/services/llm-service"
    schedule:
      interval: "weekly"
    open-pull-requests-limit: 10

  - package-ecosystem: "npm"
    directory: "/services/approval-dashboard/backend"
    schedule:
      interval: "weekly"
    open-pull-requests-limit: 10
```

### Penetration Testing

**Regular Security Testing Schedule**:
- **Weekly**: Automated vulnerability scans
- **Monthly**: Dependency updates and security patches
- **Quarterly**: Professional penetration testing
- **Annually**: Comprehensive security audit

---

## Security Audit Checklist

### Infrastructure Security

- [ ] All nodes are running latest security patches
- [ ] SSH access is restricted to bastion hosts
- [ ] Root login is disabled
- [ ] Firewall rules follow least privilege
- [ ] Security groups/network ACLs are properly configured
- [ ] Kubernetes version is up to date
- [ ] etcd is encrypted and access is restricted
- [ ] API server is not publicly accessible
- [ ] Audit logging is enabled

### Container Security

- [ ] All images are from trusted sources
- [ ] Images are scanned for vulnerabilities
- [ ] Images are signed and verified
- [ ] Containers run as non-root users
- [ ] Read-only root filesystems are used
- [ ] No privileged containers
- [ ] Resource limits are set
- [ ] Security contexts are properly configured

### Network Security

- [ ] Network policies are in place
- [ ] Default deny-all policy exists
- [ ] TLS/SSL is enforced for all external traffic
- [ ] mTLS is enabled for service-to-service communication
- [ ] Ingress uses strong TLS configuration
- [ ] Certificate expiration monitoring is in place

### Access Control

- [ ] RBAC is properly configured
- [ ] Service accounts follow least privilege
- [ ] User access is regularly reviewed
- [ ] MFA is enabled for critical accounts
- [ ] API authentication is enforced
- [ ] Authorization checks are in place

### Secret Management

- [ ] Secrets are not hardcoded in code
- [ ] Kubernetes secrets encryption is enabled
- [ ] External secret management is used (production)
- [ ] Secrets are rotated regularly
- [ ] Access to secrets is logged

### Application Security

- [ ] Input validation is implemented
- [ ] SQL injection protection is in place
- [ ] XSS protection is enabled
- [ ] CSRF protection is implemented
- [ ] Rate limiting is configured
- [ ] Security headers are set
- [ ] Dependencies are up to date
- [ ] Vulnerability scanning is automated

### Monitoring and Logging

- [ ] Security events are logged
- [ ] Logs are centralized and retained
- [ ] Alerts are configured for security events
- [ ] Audit logs are immutable
- [ ] Log access is restricted

### Backup and Recovery

- [ ] Regular backups are performed
- [ ] Backups are encrypted
- [ ] Backup restoration is tested
- [ ] Disaster recovery plan exists
- [ ] RTO/RPO are defined and met

---

## Incident Response

### Security Incident Severity

| Level | Description | Response Time | Example |
|-------|-------------|---------------|---------|
| Critical | Data breach, system compromise | Immediate | Database dump stolen, admin account compromised |
| High | Unauthorized access attempt | < 1 hour | Multiple failed login attempts, suspicious activity |
| Medium | Vulnerability discovered | < 4 hours | Unpatched CVE, misconfiguration found |
| Low | Policy violation | < 24 hours | Weak password, missing security header |

### Incident Response Procedure

**1. Detection and Identification**
```bash
# Check for suspicious activity
kubectl logs -n ai-system -l app=llm-service | grep -i "unauthorized\|failed\|forbidden"

# Check audit logs
kubectl logs -n kube-system -l component=kube-apiserver | grep -i "forbidden\|unauthorized"

# Check for unusual network activity
kubectl exec -n ai-system <pod> -- netstat -an | grep ESTABLISHED
```

**2. Containment**
```bash
# Isolate compromised pod
kubectl label pod <pod-name> -n ai-system quarantine=true
kubectl patch networkpolicy default-deny-all -n ai-system --type=merge -p '
{
  "spec": {
    "podSelector": {
      "matchLabels": {
        "quarantine": "true"
      }
    },
    "policyTypes": ["Ingress", "Egress"]
  }
}'

# Revoke access
kubectl delete rolebinding <suspicious-binding> -n ai-system

# Disable compromised user account
# In application database:
UPDATE users SET active = false WHERE username = 'compromised-user';
```

**3. Investigation**
```bash
# Collect evidence
kubectl logs <pod-name> -n ai-system --previous > evidence-logs.txt
kubectl describe pod <pod-name> -n ai-system > evidence-pod.txt
kubectl get events -n ai-system --sort-by='.lastTimestamp' > evidence-events.txt

# Analyze audit logs
# Export from Elasticsearch/Kibana
# Look for patterns and timeline
```

**4. Eradication**
```bash
# Remove malicious code
kubectl delete pod <compromised-pod> -n ai-system

# Rotate compromised credentials
kubectl delete secret <compromised-secret> -n ai-system
kubectl create secret generic <secret-name> --from-literal=key=new-value -n ai-system

# Patch vulnerabilities
kubectl set image deployment/<deployment> container=<new-image> -n ai-system
```

**5. Recovery**
```bash
# Restore from backup if needed
# See Backup and Recovery section

# Restart services
kubectl rollout restart deployment/<deployment> -n ai-system

# Verify system integrity
kubectl get pods -n ai-system
# Test endpoints
```

**6. Post-Incident Review**
- Document incident timeline
- Identify root cause
- Update security controls
- Share lessons learned

### Security Contacts

- **Security Team**: security@example.com
- **Incident Response**: #security-incidents
- **On-Call Security Engineer**: security-oncall@example.com
- **CISO**: ciso@example.com

---

## Conclusion

Security is an ongoing process, not a one-time task. Regular reviews, updates, and testing are essential to maintain a secure system.

**Key Takeaways**:
- Implement defense in depth
- Follow least privilege principle
- Keep systems updated
- Monitor and log everything
- Test security controls regularly
- Have an incident response plan

For security concerns or to report vulnerabilities, contact: security@example.com

---

**Last Updated**: 2025-01-15
**Version**: 1.0.0
