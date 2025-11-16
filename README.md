# 🚀 Deployment MicroK8s - eComove

Guía completa de instalación, configuración y ejecución de la arquitectura de microservicios eComove en MicroK8s con Vault para gestión de secretos.

---

## 📋 Tabla de Contenidos

1. [Arquitectura General](#arquitectura-general)
2. [Requisitos Previos](#requisitos-previos)
3. [Configuración de Vault](#configuración-de-vault)
4. [Estructura de Carpetas](#estructura-de-carpetas)
5. [Instrucciones de Despliegue](#instrucciones-de-despliegue)
6. [Verificación y Troubleshooting](#verificación-y-troubleshooting)

---

## 🏗️ Arquitectura General

```
┌─────────────────────────────────────────────────────────────┐
│                    VAULT EXTERNO (192.168.1.18)              │
│                    Puerto: 8200                              │
└─────────────────────────────────────────────────────────────┘
         ▲                                      ▲
         │                                      │
         │                                      │
┌────────┴────────────────────────┬────────────┴──────┐
│                                 │                   │
│                                 │                   │
▼                                 ▼                   ▼
┌──────────────────────────┐  ┌──────────────────┐ ┌──────────────┐
│  development-ecomove     │  │   postgres       │ │  Otros NS    │
│  Namespace               │  │  Namespace       │ │              │
│                          │  │                  │ │              │
│ ┌────────────────────┐   │  │ ┌──────────────┐ │ │              │
│ │ ecomove-back Pod   │   │  │ │ postgres-0   │ │ │              │
│ │ (Deployment x2)    │   │  │ │ (StatefulSet)│ │ │              │
│ │                    │   │  │ │              │ │ │              │
│ │ ServiceAccount:    │   │  │ │ ServiceAcct: │ │ │              │
│ │ myapp-sa           │   │  │ │ myapp-bd     │ │ │              │
│ │                    │   │  │ │              │ │ │              │
│ │ Role: vehicles     │   │  │ │ Role:        │ │ │              │
│ └────────────────────┘   │  │ │ bd-vehicles  │ │ │              │
│                          │  │ │              │ │ │              │
│ ┌────────────────────┐   │  │ │ ┌──────────┐ │ │ │              │
│ │ external-vault     │   │  │ │ │external- │ │ │ │              │
│ │ (ExternalName Svc) │   │  │ │ │vault-bd  │ │ │ │              │
│ │ ▶ 192.168.1.18:8200│   │  │ │ │ Endpoints│ │ │ │              │
│ └────────────────────┘   │  │ │ │ 192.168..│ │ │ │              │
│                          │  │ │ └──────────┘ │ │ │              │
│ Service: ecomove-svc     │  │ │ Service:     │ │ │              │
│ Port: 3000               │  │ │ postgres     │ │ │              │
│                          │  │ │ Port: 5432   │ │ │              │
└──────────────────────────┘  └──────────────────┘ └──────────────┘
```

---

## 📌 Requisitos Previos

### Hardware y Software
- **MicroK8s** v1.20+ instalado y ejecutándose
- **kubectl** configurado
- **Vault** 1.20.4+ ejecutándose en `192.168.1.18:8200`
- **Helm** (opcional, para Vault Injector)
- **Docker** para construir imágenes

### Verificar MicroK8s
```bash
microk8s status
microk8s kubectl version
```

---

## 🔐 Configuración de Vault

### 1. Habilitar Kubernetes Auth en Vault

```bash
# Conectar a Vault
export VAULT_ADDR="http://192.168.1.18:8200"
export VAULT_TOKEN="tu-token-root"

# Habilitar Kubernetes auth method
vault auth enable kubernetes

# Configurar Kubernetes auth
vault write auth/kubernetes/config \
  kubernetes_host="https://<MICROK8S_API_IP>:16443" \
  kubernetes_ca_cert=@/path/to/ca.crt \
  token_reviewer_jwt="@/path/to/token"
```

### 2. Crear Secretos en Vault

```bash
# Secreto para Backend (vehicles)
vault kv put secret/ecomove/vehicles \
  DATABASE_URL="postgresql://app_user:tu_password@postgres.postgres.svc.cluster.local:5432/ecomove" \
  URL_IMAGES="http://ecomove-svc-images:3000"

# Secreto para Base de Datos (bd-vehicles)
vault kv put secret/ecomove/bd-vehicles \
  DB_USER="app_user" \
  POSTGRES_PASSWORD="swywswproject2025" \
  POSTGRES_DB="ecomove" \
  DB_APP_USER_PASSWORD="eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9"
```

### 3. Crear Políticas en Vault

#### Política para Backend (vehicles)
```bash
# Crear archivo policy-vehicles.hcl
cat > policy-vehicles.hcl << 'EOF'
path "secret/data/ecomove/vehicles" {
  capabilities = ["read", "list"]
}

path "secret/metadata/ecomove/vehicles" {
  capabilities = ["read", "list"]
}
EOF

vault policy write vehicles policy-vehicles.hcl
```

#### Política para Base de Datos (bd-vehicles)
```bash
# Crear archivo policy-bd-vehicles.hcl
cat > policy-bd-vehicles.hcl << 'EOF'
path "secret/data/ecomove/bd-vehicles" {
  capabilities = ["read", "list"]
}

path "secret/metadata/ecomove/bd-vehicles" {
  capabilities = ["read", "list"]
}
EOF

vault policy write bd-vehicles policy-bd-vehicles.hcl
```

### 4. Crear Roles en Vault

#### Rol para Backend (namespace: development-ecomove)
```bash
vault write auth/kubernetes/role/vehicles \
  bound_service_account_names=myapp-sa \
  bound_service_account_namespaces=development-ecomove \
  policies=vehicles \
  ttl=24h
```

#### Rol para Base de Datos (namespace: postgres)
```bash
vault write auth/kubernetes/role/bd-vehicles \
  bound_service_account_names=myapp-bd \
  bound_service_account_namespaces=postgres \
  policies=bd-vehicles \
  ttl=24h
```

---

## 📁 Estructura de Carpetas

### `k8s-vault/`
Configuración de Vault y conectores entre namespaces.

| Archivo | Propósito |
|---------|-----------|
| `external-vault.yml` | Service externo en `development-ecomove` que apunta a Vault (192.168.1.18:8200) |
| `external-bd.yml` | Service externo en `postgres` que apunta a Vault (192.168.1.18:8200) |
| `internal-app.yml` | ServiceAccount `myapp-sa` en `development-ecomove` para autenticación con Vault |
| `internal-bd.yml` | ServiceAccount `myapp-bd` en `postgres` para autenticación con Vault |
| `service.yml` | **ExternalName Service** en `postgres` que redirecciona a `external-vault` de `development-ecomove` |

### `k8s-postgrest/`
Configuración de PostgreSQL como StatefulSet.

| Archivo | Propósito |
|---------|-----------|
| `ns.yml` | Crea namespace `postgres` |
| `postgres-stateful.yml` | StatefulSet con Vault Agent injection para obtener secretos |
| `postgres-pvc.yml` | PersistentVolumeClaim para datos de PostgreSQL |
| `postgres-pv.yml` | PersistentVolume para almacenamiento |
| `service.yml` | Service ClusterIP para exponer PostgreSQL (puerto 5432) |
| `secrets-posgrets.yml` | Secrets K8s de respaldo (alternativo a Vault) |

### `k8s-backend/`
Configuración del Backend (API eComove).

| Archivo | Propósito |
|---------|-----------|
| `deployment.yml` | Deployment de 2 réplicas con inyección de Vault |
| `service.yml` | Service ClusterIP en puerto 3000 |
| `secrets.yml` | Secrets K8s de respaldo |
| `migration-job.yml` | Job para ejecutar migraciones de BD |
| `namespace.yml` | Namespace `development-ecomove` |

### `k8s-front-ecomove/`
Configuración del Frontend.

| Archivo | Propósito |
|---------|-----------|
| `deployment.yml` | Deployment del Frontend |
| `service.yml` | Service para exponer Frontend |
| `ingress.yml` | Ingress para rutas HTTP/HTTPS |
| `secrets.yml` | Variables de entorno del Frontend |

### `k8s-backend-images/` y `k8s-posgrest-images/`
Versiones alternativas con inyección de imágenes personalizadas.

---

## 🚀 Instrucciones de Despliegue

### Paso 1: Crear Namespaces

```bash
# Namespace para PostgreSQL
kubectl apply -f k8s-postgrest/ns.yml

# Namespace para Backend (si no existe en k8s-backend)
kubectl create namespace development-ecomove
```

### Paso 2: Configurar Vault Services

```bash
# Service externo en development-ecomove
kubectl apply -f k8s-vault/external-vault.yml

# Service externo en postgres namespace
kubectl apply -f k8s-vault/external-bd.yml

# Service que conecta postgres con development-ecomove
kubectl apply -f k8s-vault/service.yml

# ServiceAccounts
kubectl apply -f k8s-vault/internal-app.yml
kubectl apply -f k8s-vault/internal-bd.yml
```

### Paso 3: Instalar Vault Agent Injector (Helm)

```bash
# Agregar repositorio de Vault
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update

# Instalar Vault Agent Injector
helm install vault hashicorp/vault \
  --namespace vault \
  --create-namespace \
  --values vault-values.yaml

# O instalar solo el Injector Controller
helm install vault-injector hashicorp/vault \
  --namespace vault \
  --create-namespace \
  --set injector.enabled=true \
  --set server.enabled=false
```

### Paso 4: Desplegar PostgreSQL

```bash
# PersistentVolume y PersistentVolumeClaim
kubectl apply -f k8s-postgrest/postgres-pv.yml
kubectl apply -f k8s-postgrest/postgres-pvc.yml

# Service de PostgreSQL
kubectl apply -f k8s-postgrest/service.yml

# StatefulSet con Vault Agent
kubectl apply -f k8s-postgrest/postgres-stateful.yml

# Verificar
kubectl get pods -n postgres
kubectl logs -f pod/postgres-0 -n postgres
```

### Paso 5: Desplegar Backend

```bash
# Secrets (backup, si Vault no está disponible)
kubectl apply -f k8s-backend/secrets.yml

# Service del Backend
kubectl apply -f k8s-backend/service.yml

# Deployment con Vault Agent
kubectl apply -f k8s-backend/deployment.yml

# Verificar
kubectl get pods -n development-ecomove
kubectl logs -f deployment/ecomove-back -n development-ecomove
```

### Paso 6: Ejecutar Migraciones (Opcional)

```bash
# Job de migración
kubectl apply -f k8s-backend/migration-job.yml

# Ver logs
kubectl logs -f job/migration-job -n development-ecomove
```

### Paso 7: Desplegar Frontend

```bash
kubectl apply -f k8s-front-ecomove/
kubectl get svc -n front
```

---

## ✅ Verificación y Troubleshooting

### Verificar que todo está corriendo

```bash
# Ver todos los namespaces
kubectl get namespaces

# Ver todos los pods
kubectl get pods --all-namespaces

# Ver servicios
kubectl get svc --all-namespaces

# Ver endpoints (para servicios externos)
kubectl get endpoints -n postgres
kubectl get endpoints -n development-ecomove
```

### Verificar conectividad con Vault

```bash
# Desde pod de PostgreSQL
kubectl exec -it pod/postgres-0 -n postgres -- \
  curl -v http://external-vault-bd.postgres.svc.cluster.local:8200/v1/sys/health

# Desde pod del Backend
kubectl exec -it deployment/ecomove-back -n development-ecomove -- \
  curl -v http://external-vault.development-ecomove.svc.cluster.local:8200/v1/sys/health
```

### Resolver problemas de DNS

```bash
# Dentro de un pod, probar DNS
kubectl exec -it pod/postgres-0 -n postgres -- nslookup external-vault-bd.postgres.svc.cluster.local

# Ver DNS del cluster
kubectl get svc -n kube-system -l k8s-app=kube-dns
```

### Ver logs de Vault Agent

```bash
# PostgreSQL
kubectl logs -f pod/postgres-0 -n postgres -c vault-agent-init

# Backend
kubectl logs -f deployment/ecomove-back -n development-ecomove -c vault-agent
```

### Problemas comunes

#### Error: "dial tcp: lookup external-vault on X.X.X.X:53: server misbehaving"
**Causa**: El pod no resuelve el nombre DNS del servicio.

**Solución**:
```bash
# Verificar que el Endpoint existe
kubectl get endpoints -n postgres

# Verificar que el Service existe
kubectl get svc -n postgres

# Verificar la configuración de DNS del cluster
kubectl get configmap coredns -n kube-system -o yaml

# Reiniciar coredns si es necesario
kubectl rollout restart deployment/coredns -n kube-system
```

#### Error: "failed to authenticate: vault login failed"
**Causa**: El ServiceAccount no tiene permiso en Vault o la política no existe.

**Solución**:
```bash
# Verificar roles en Vault
vault list auth/kubernetes/role

# Verificar política
vault policy read bd-vehicles

# Reiniciar el pod para forzar autenticación
kubectl delete pod/postgres-0 -n postgres
```

#### Pod no obtiene secretos de Vault
**Causa**: Vault Agent injection no está funcionando.

**Solución**:
```bash
# Verificar que vault-injector está corriendo
kubectl get pods -n vault

# Ver los logs del injector
kubectl logs -f deployment/vault-injector -n vault

# Verificar anotaciones del pod
kubectl get pod postgres-0 -n postgres -o yaml | grep vault
```

---

## 🔑 Referencia de Comandos Útiles

```bash
# Ver secretos en Vault desde el cluster
kubectl exec -it pod/postgres-0 -n postgres -- cat /vault/secrets/db.env

# Acceder a PostgreSQL desde otro pod
kubectl run -it --rm debug --image=postgres:16.3 --restart=Never -n postgres -- \
  psql -h postgres.postgres.svc.cluster.local -U postgres -d ecomove

# Port-forward para acceder a servicios localmente
kubectl port-forward svc/ecomove-svc 3000:3000 -n development-ecomove

# Ver eventos de los pods
kubectl describe pod postgres-0 -n postgres

# Ver YAML renderizado de un deployment
kubectl get deployment ecomove-back -n development-ecomove -o yaml
```

---

## 🛠️ Configuración de RBAC (Role-Based Access Control)

Si necesitas RBAC adicional:

```bash
# Crear Role para Vault
cat << 'EOF' | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: vault-auth
rules:
- apiGroups: [""]
  resources: ["serviceaccounts/token"]
  verbs: ["create"]
- apiGroups: [""]
  resources: ["serviceaccounts"]
  verbs: ["get"]
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get"]
EOF

# Bind Role a ServiceAccount
kubectl create clusterrolebinding vault-auth \
  --clusterrole=vault-auth \
  --serviceaccount=vault:vault
```

---

## 📚 Enlaces Útiles

- [Vault Documentation](https://www.vaultproject.io/docs)
- [Kubernetes Auth Method](https://www.vaultproject.io/docs/auth/kubernetes)
- [Vault Agent Injector](https://www.vaultproject.io/docs/platform/k8s/injector)
- [MicroK8s Documentation](https://microk8s.io/docs)

---

## ⚠️ Notas de Seguridad

- **No guardes tokens en archivos de configuración** (usa Vault)
- **Cifra los secretos en etcd**: `kubectl edit -f kubeadm-config.yaml`
- **Limita acceso a RBAC**: Solo los ServiceAccounts necesarios
- **Usa TLS para Vault**: En producción, usa `https://`
- **Rotación de secretos**: Implementa una estrategia de rotación periódica

---

**Última actualización**: 16 de Noviembre de 2025
**Autor**: eComove DevOps Team