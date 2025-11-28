# k8s-backend

📌 Descripción

Este directorio contiene los manifiestos de Kubernetes para desplegar el backend principal (ecomove-back) del servicio "vehicles". Incluye:

- `namespace.yml` — Namespace `development-ecomove` donde se desplegará el backend.
- `service.yml` — Service tipo `ClusterIP` que expone el puerto 3000 dentro del cluster.
- `deployment.yml` — Deployment para la aplicación `ecomove-back` (imagen: `jdavidperalta/backend-pipeline:0.0.2`). Contiene anotaciones para inyección de secretos desde Vault y utiliza el `serviceAccount` `myapp-sa`.
- `migration-job.yml` — Job para correr migraciones con drizzle (npx drizzle-kit push). También inyecta secretos desde Vault.

---

## Requisitos previos

- El `namespace` `development-ecomove` no existe aún en el cluster (se crea aquí).
- `k8s-vault` debe estar desplegado primero (servicio `external-vault` y Endpoints configurados), y las rutas y secretos en Vault deben estar disponibles: `secret/data/ecomove/vehicles`.
- Debe existir el `ServiceAccount` `myapp-sa` (se encuentra en `k8s-vault/internal-app.yml`).
- Asegúrese de que la base de datos Postgres para `vehicles` esté desplegada y accesible (ver `k8s-postgrest`).

---

## Orden de despliegue recomendado

1. Aplicar recursos de Vault y ServiceAccounts (desde `k8s-vault`):

```bash
kubectl apply -f k8s-vault/internal-app.yml
kubectl apply -f k8s-vault/internal-bd.yml
kubectl apply -f k8s-vault/external-vault.yml
kubectl apply -f k8s-vault/external-bd.yml
```

2. Desplegar la base de datos de vehículos (`k8s-postgrest`) y persistencia (PV/PVC/StatefulSet):

```bash
kubectl apply -f k8s-postgrest/ns.yml
kubectl apply -f k8s-postgrest/posgrest-pv.yml
kubectl apply -f k8s-postgrest/postgres-pvc.yml
kubectl apply -f k8s-postgrest/postgres-stateful.yml
kubectl apply -f k8s-postgrest/service.yml
```

3. Desplegar el `namespace` y el `deployment` del backend (solo aplicar `namespace.yml` primero si aún no existe):

```bash
kubectl apply -f k8s-backend/namespace.yml
kubectl apply -f k8s-backend/service.yml
```

4. Correr la migración (opcional o según flujo de CI):

```bash
kubectl apply -f k8s-backend/migration-job.yml
# Revisar logs
kubectl logs job/drizzle-migrate -n development-ecomove
```

5. Finalmente desplegar el `deployment` del backend:

```bash
kubectl apply -f k8s-backend/deployment.yml
```

---

## Verificación y debug

- Verificar estado del pod:

```bash
kubectl get pods -n development-ecomove -l app=ecomove
kubectl describe pod <pod-name> -n development-ecomove
kubectl logs <pod-name> -n development-ecomove
```

- Comprobar que el servicio exista y apunte correctamente:

```bash
kubectl get svc -n development-ecomove ecomove-svc
```

---

## Notas y consideraciones

- El `deployment.yml` incluye anotaciones para la inyección de secretos desde Vault (agent-inject). Verificar que `vault` esté accesible y tenga los secretos esperados.
- Si usas MicroK8s, reemplaza `kubectl` por `microk8s kubectl`.
- Asegúrate de que `myapp-sa` y `myapp-bd` se creen con sus respectivas políticas de RBAC si tu cluster necesita permisos específicos.
