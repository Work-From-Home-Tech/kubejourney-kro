# WordPress Server Resource Graph Definition (RGD)

A Kubernetes Resource Graph Definition (RGD) for deploying WordPress applications with MariaDB backend using the [kro](https://kro.run) operator.

## Overview

This RGD provides a simple, declarative way to deploy complete WordPress stacks including:
- WordPress frontend deployment
- MariaDB database backend
- Kubernetes services for both components
- Optional ingress configuration
- Support for custom namespaces

## Features

- **Multi-namespace Support**: Deploy WordPress instances in any namespace
- **Configurable Replicas**: Scale WordPress frontend pods
- **Custom Database Passwords**: Secure database configuration
- **Ingress Integration**: Optional external access configuration
- **Service Discovery**: Automatic database connection configuration
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

## Usage Examples

### Basic Deployment (Default Namespace)

```yaml
apiVersion: kro.run/v1alpha1
kind: WordpressServer
metadata:
  name: my-wordpress
spec:
  name: wordpress1
  ingress:
    enabled: true
```

**Result:**
- Resources created in: `default` namespace
- Database connection: `wordpress1-service-db.default.svc:3306`
- Single replica WordPress deployment

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
```

**Result:**
- Resources created in: `development` namespace
- Database connection: `wordpress-dev-service-db.development.svc:3306`
- Two replica WordPress deployment with custom password

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
```

### 4. Access WordPress

If ingress is enabled, access via the configured hostname. Otherwise, use port-forwarding:

```bash
kubectl port-forward -n your-namespace service/your-wordpress-name-service 8080:80
```

Then visit: http://localhost:8080

## Architecture

The RGD creates the following Kubernetes resources:

### Frontend (WordPress)
- **Deployment**: Runs WordPress containers
- **Service**: ClusterIP service exposing port 80
- **ConfigMap**: Environment variables for database connection

### Backend (MariaDB)
- **Deployment**: Runs MariaDB container
- **Service**: ClusterIP service exposing port 3306
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

- All resources (frontend, backend, services) are created in the same namespace as the WordpressServer instance
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

### Debugging Commands

```bash
# Check RGD status
kubectl get resourcegraphdefinition wordpress

# Check instance status
kubectl get wordpressserver -A

# View detailed instance information
kubectl describe wordpressserver -n your-namespace instance-name

# Check all resources created by instance
kubectl get all -n your-namespace -l kro.run/instance=instance-name
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
3. Verify backward compatibility
4. Update this README with any new features

## License

This project is provided as-is for educational and development purposes.
