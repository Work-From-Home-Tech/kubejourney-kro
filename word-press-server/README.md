# WordPress Server Resource Graph Definition (RGD)

A Kubernetes Resource Graph Definition (RGD) for deploying WordPress applications with MariaDB backend using the [kro](https://kro.run) operator.

## Overview

This RGD provides a simple, declarative way to deploy complete WordPress stacks including:
- WordPress frontend deployment with persistent storage
- MariaDB database backend with persistent storage
- Kubernetes services for both components
- Optional ingress configuration
- Support for custom namespaces
- Configurable persistent storage with automatic cleanup

## Features

- **Persistent Storage**: Data persists across pod restarts and deployments
- **Multi-namespace Support**: Deploy WordPress instances in any namespace
- **Configurable Replicas**: Scale WordPress frontend pods
- **Custom Database Passwords**: Secure database configuration
- **Ingress Integration**: Optional external access configuration
- **Service Discovery**: Automatic database connection configuration
- **Storage Flexibility**: Configurable storage classes and sizes
- **Automatic Cleanup**: Storage resources are automatically cleaned up when instances are deleted
- **Backward Compatibility**: Works with existing deployments

## Quick Start

### 1. Prerequisites

Ensure you have:
- Kubernetes cluster with kro operator installed
- kubectl configured to access your cluster

### 2. Apply the Resource Graph Definition

```bash
kubectl apply -f wordpress-rgd.yaml
```

### 3. Create a WordPress Instance

```bash
kubectl apply -f wp-instance.yaml
```

## Configuration Reference

### WordpressServer Spec

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `name` | string | *required* | Base name for all resources |
| `namespace` | string | `"default"` | Target namespace for service discovery |
| `image` | string | `"wordpress:6.4-apache"` | WordPress container image |
| `replicas` | integer | `1` | Number of WordPress frontend pods |
| `db_password` | string | `"my-secret-pw"` | MariaDB root password |
| `ingress.enabled` | boolean | `false` | Enable ingress resource creation |
| `ingress.host` | string | *auto-generated* | Ingress hostname |
| `ingress.port` | integer | `80` | Ingress target port |
| `storage.enabled` | boolean | `true` | Enable persistent storage |
| `storage.storageClass` | string | `"local-path"` | Storage class for PVs and PVCs |
| `storage.wordpress.size` | string | `"10Gi"` | WordPress persistent volume size |
| `storage.mariadb.size` | string | `"20Gi"` | MariaDB persistent volume size |

## Persistent Storage

### Overview
The WordPress RGD includes comprehensive persistent storage support for both WordPress and MariaDB components, ensuring data persistence across pod restarts, deployments, and scaling operations.

### Storage Architecture

The RGD creates the following storage resources when `storage.enabled` is true (default):

#### PersistentVolumes (PVs)
- **WordPress PV**: `{instance-name}-wordpress-pv`
  - Size: 10Gi (configurable via `storage.wordpress.size`)
  - Access Mode: ReadWriteOnce
  - Storage Class: local-path (configurable via `storage.storageClass`)
  - Host Path: `/tmp/{instance-name}-wordpress-data`

- **MariaDB PV**: `{instance-name}-mariadb-pv`
  - Size: 20Gi (configurable via `storage.mariadb.size`)
  - Access Mode: ReadWriteOnce
  - Storage Class: local-path (configurable via `storage.storageClass`)
  - Host Path: `/tmp/{instance-name}-mariadb-data`

#### PersistentVolumeClaims (PVCs)
- **WordPress PVC**: `{instance-name}-wordpress-pvc`
- **MariaDB PVC**: `{instance-name}-mariadb-pvc`

### Storage Benefits

#### Data Persistence
- WordPress files, themes, plugins, and uploads persist across pod restarts
- MariaDB database data persists across pod restarts
- No data loss during deployments or scaling operations

#### Configurable Storage
- Users can specify custom storage sizes for both components
- Storage class can be customized per deployment
- Default values provide sensible starting points

#### Scalability
- Each WordPress instance gets its own dedicated storage
- Storage is properly namespaced and isolated
- Supports multiple environments (dev, staging, prod)

### Storage Cleanup
- **Automatic Cleanup**: PersistentVolumes use `Delete` reclaim policy
- When a WordPress instance is deleted, all associated storage is automatically removed
- No manual cleanup required for PVs and PVCs
- Data directories on the host are also cleaned up automatically

## Usage Examples

### Basic Deployment (Default Storage)

```yaml
apiVersion: kro.run/v1alpha1
kind: WordpressServer
metadata:
  name: my-wordpress
spec:
  name: wordpress1
  ingress:
    enabled: true
  # Storage will use defaults: enabled=true, 10Gi WordPress, 20Gi MariaDB, local-path storage class
```

**Result:**
- Resources created in: `default` namespace
- Database connection: `wordpress1-service-db.default.svc:3306`
- Single replica WordPress deployment with persistent storage
- WordPress storage: 10Gi, MariaDB storage: 20Gi

### Custom Storage Configuration

```yaml
apiVersion: kro.run/v1alpha1
kind: WordpressServer
metadata:
  name: my-wordpress-custom
spec:
  name: my-site
  storage:
    storageClass: "fast-ssd"
    wordpress:
      size: "50Gi"
    mariadb:
      size: "100Gi"
  ingress:
    enabled: true
```

**Result:**
- Custom storage class: `fast-ssd`
- WordPress storage: 50Gi
- MariaDB storage: 100Gi
- All other settings use defaults

### Disable Storage (Legacy Mode)

```yaml
apiVersion: kro.run/v1alpha1
kind: WordpressServer
metadata:
  name: my-wordpress-no-storage
spec:
  name: wordpress-temp
  storage:
    enabled: false
  ingress:
    enabled: true
```

**Result:**
- WordPress uses `emptyDir` volume (data lost on pod restart)
- MariaDB has no persistent storage (data lost on pod restart)
- Useful for testing or temporary deployments

### Custom Namespace Deployment

```yaml
apiVersion: kro.run/v1alpha1
kind: WordpressServer
metadata:
  name: my-wordpress-dev
  namespace: development
spec:
  name: wordpress-dev
  namespace: development
  ingress:
    enabled: true
  db_password: "dev-secret-password"
  replicas: 2
  storage:
    wordpress:
      size: "20Gi"
    mariadb:
      size: "40Gi"
```

**Result:**
- Resources created in: `development` namespace
- Database connection: `wordpress-dev-service-db.development.svc:3306`
- Two replica WordPress deployment with custom password
- Custom storage sizes: WordPress 20Gi, MariaDB 40Gi

### Production Configuration

```yaml
apiVersion: kro.run/v1alpha1
kind: WordpressServer
metadata:
  name: wordpress-prod
  namespace: production
spec:
  name: wordpress-prod
  namespace: production
  image: "wordpress:6.4-apache"
  replicas: 3
  db_password: "super-secure-production-password"
  ingress:
    enabled: true
    host: "myblog.example.com"
    port: 80
  storage:
    storageClass: "premium-ssd"
    wordpress:
      size: "100Gi"
    mariadb:
      size: "200Gi"
```

## Deployment Steps

### 1. Create Target Namespace (if using custom namespace)

```bash
kubectl create namespace development
```

### 2. Deploy WordPress Instance

```bash
kubectl apply -f your-wordpress-instance.yaml
```

### 3. Verify Deployment

```bash
# Check all resources
kubectl get all -n your-namespace

# Check WordPress instance status
kubectl get wordpressserver -n your-namespace

# Verify database connection
kubectl describe pod -n your-namespace -l app=your-wordpress-name | grep WORDPRESS_DB_HOST

# Check storage resources
kubectl get pv,pvc -n your-namespace
```

### 4. Access WordPress

If ingress is enabled, access via the configured hostname. Otherwise, use port-forwarding:

```bash
kubectl port-forward -n your-namespace service/your-wordpress-name-service 8080:80
```

Then visit: http://localhost:8080

## Architecture

The RGD creates the following Kubernetes resources:

### Storage Layer (when storage.enabled=true)
- **PersistentVolumes**: Host-path based storage for WordPress and MariaDB
- **PersistentVolumeClaims**: Storage requests bound to PVs
- **Storage Paths**: 
  - WordPress data: `/tmp/{instance-name}-wordpress-data`
  - MariaDB data: `/tmp/{instance-name}-mariadb-data`

### Frontend (WordPress)
- **Deployment**: Runs WordPress containers with persistent volume mounted at `/var/www/html`
- **Service**: ClusterIP service exposing port 80
- **Storage**: PVC-backed persistent storage for WordPress files

### Backend (MariaDB)
- **Deployment**: Runs MariaDB container with persistent volume mounted at `/var/lib/mysql`
- **Service**: ClusterIP service exposing port 3306
- **Storage**: PVC-backed persistent storage for database files
- **Environment**: Configured with database name and credentials

### Networking
- **Ingress** (optional): External access configuration
- **Service Discovery**: Automatic DNS resolution between components

## Namespace Support

### How It Works

The namespace field in the spec controls service discovery for database connections:

```yaml
env:
  - name: WORDPRESS_DB_HOST
    value: ${schema.spec.name}-service-db.${schema.spec.namespace}.svc:3306
```

### Benefits

1. **Multi-tenancy**: Deploy multiple WordPress instances in different namespaces
2. **Resource Isolation**: Better separation between environments
3. **Environment Separation**: Easy dev/staging/production organization
4. **Team Isolation**: Different teams can use different namespaces
5. **Backward Compatibility**: Existing deployments continue working

### Cross-Namespace Considerations

- All resources (frontend, backend, services, storage) are created in the same namespace as the WordpressServer instance
- The `namespace` field in the spec is used for service discovery, not resource placement
- Ensure the target namespace exists before deploying

## Troubleshooting

### Common Issues

1. **Namespace doesn't exist**
   ```bash
   kubectl create namespace your-namespace
   ```

2. **Database connection issues**
   - Verify the database service exists: `kubectl get svc -n your-namespace`
   - Check database pod logs: `kubectl logs -n your-namespace -l app=your-name-db`

3. **WordPress pods not starting**
   - Check pod events: `kubectl describe pod -n your-namespace pod-name`
   - Verify image availability and resource limits

4. **Ingress not working**
   - Ensure ingress controller is installed
   - Check ingress resource: `kubectl get ingress -n your-namespace`

5. **Storage issues**
   - Check PV status: `kubectl get pv | grep your-instance-name`
   - Check PVC status: `kubectl get pvc -n your-namespace`
   - Verify storage class exists: `kubectl get storageclass`
   - Check host path permissions: Ensure `/tmp/{instance-name}-*-data` directories are accessible

6. **Data not persisting**
   - Verify storage is enabled: Check your WordpressServer spec has `storage.enabled: true`
   - Check volume mounts: `kubectl describe pod -n your-namespace pod-name`
   - Verify PVC is bound: `kubectl get pvc -n your-namespace`

### Storage Verification

```bash
# Check storage resources
kubectl get pv,pvc -n your-namespace

# Verify volume mounts
kubectl describe pod -n your-namespace -l app=your-instance-name

# Check storage usage
kubectl exec -n your-namespace deployment/your-instance-name -- df -h /var/www/html
kubectl exec -n your-namespace deployment/your-instance-name-db -- df -h /var/lib/mysql

# Test data persistence
kubectl exec -n your-namespace deployment/your-instance-name -- touch /var/www/html/test-file
kubectl delete pod -n your-namespace -l app=your-instance-name
# Wait for pod to restart, then check if file exists
kubectl exec -n your-namespace deployment/your-instance-name -- ls /var/www/html/test-file
```

### Debugging Commands

```bash
# Check RGD status
kubectl get resourcegraphdefinition wordpress

# Check instance status
kubectl get wordpressserver -A

# View detailed instance information
kubectl describe wordpressserver -n your-namespace instance-name

# Check all resources created by instance
kubectl get all,pv,pvc -n your-namespace -l kro.run/instance=instance-name

# Check storage-specific resources
kubectl get pv | grep your-instance-name
kubectl get pvc -n your-namespace | grep your-instance-name
```

## File Structure

```
.
├── README.md                          # This documentation
├── wordpress-rgd.yaml                 # Resource Graph Definition
├── wp-instance.yaml                   # Basic instance example
└── wp-instance-with-namespace.yaml    # Custom namespace example
```

## Contributing

When modifying the RGD:

1. Update the schema version if breaking changes are made
2. Test with both default and custom namespaces
3. Test with both storage enabled and disabled
4. Verify storage persistence across pod restarts
5. Verify backward compatibility
6. Update this README with any new features

## License

This project is provided as-is for educational and development purposes.
