# WordPress KRO Persistent Storage Implementation

## Overview
This document describes the persistent storage implementation added to the WordPress ResourceGraphDefinition (RGD) for both WordPress and MariaDB components.

## Changes Made

### 1. Schema Updates
Added storage configuration options to the RGD schema:
```yaml
storage:
  storageClass: string | default="local-path"
  wordpress:
    size: string | default="10Gi"
  mariadb:
    size: string | default="20Gi"
```

### 2. New Resources Added
The RGD now creates the following additional resources:

#### PersistentVolumes (PVs)
- **WordPress PV**: `${schema.spec.name}-wordpress-pv`
  - Size: 10Gi (configurable)
  - Access Mode: ReadWriteOnce
  - Storage Class: local-path
  - Host Path: `/tmp/${schema.spec.name}-wordpress-data`

- **MariaDB PV**: `${schema.spec.name}-mariadb-pv`
  - Size: 20Gi (configurable)
  - Access Mode: ReadWriteOnce
  - Storage Class: local-path
  - Host Path: `/tmp/${schema.spec.name}-mariadb-data`

#### PersistentVolumeClaims (PVCs)
- **WordPress PVC**: `${schema.spec.name}-wordpress-pvc`
- **MariaDB PVC**: `${schema.spec.name}-mariadb-pvc`

### 3. Deployment Updates

#### WordPress Deployment (frontend)
- **Before**: Used `emptyDir: {}` for temporary storage
- **After**: Uses PVC-backed persistent storage
- **Mount Path**: `/var/www/html` (unchanged)
- **Volume**: References `${schema.spec.name}-wordpress-pvc`

#### MariaDB Deployment (backend)
- **Before**: No persistent storage (data lost on pod restart)
- **After**: Uses PVC-backed persistent storage
- **Mount Path**: `/var/lib/mysql` (MariaDB data directory)
- **Volume**: References `${schema.spec.name}-mariadb-pvc`

## Benefits

### Data Persistence
- WordPress files, themes, plugins, and uploads persist across pod restarts
- MariaDB database data persists across pod restarts
- No data loss during deployments or scaling operations

### Configurable Storage
- Users can specify custom storage sizes for both components
- Storage class can be customized per deployment
- Default values provide sensible starting points

### Scalability
- Each WordPress instance gets its own dedicated storage
- Storage is properly namespaced and isolated
- Supports multiple environments (dev, staging, prod)

## Usage Examples

### Basic Usage (uses defaults)
```yaml
apiVersion: kro.run/v1alpha1
kind: WordpressServer
metadata:
  name: my-wordpress
spec:
  name: my-site
  # Storage will use defaults: 10Gi WordPress, 20Gi MariaDB, local-path storage class
```

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
```

## Resource Naming Convention
- WordPress PV: `{instance-name}-wordpress-pv`
- WordPress PVC: `{instance-name}-wordpress-pvc`
- MariaDB PV: `{instance-name}-mariadb-pv`
- MariaDB PVC: `{instance-name}-mariadb-pvc`

## Storage Paths
- WordPress data: `/tmp/{instance-name}-wordpress-data`
- MariaDB data: `/tmp/{instance-name}-mariadb-data`

## Backward Compatibility
- Existing WordPress instances will continue to work
- New storage options are optional with sensible defaults
- No breaking changes to existing API
