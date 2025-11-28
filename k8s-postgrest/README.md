# k8s-postgrest

📌 Descripción

Contiene los recursos para desplegar la base de datos PostgreSQL de la aplicación principal (`postgres`). Incluye:

- `ns.yml` — Namespace `postgres` para los recursos de Postgres.
- `posgrest-pv.yml` — PersistentVolume (hostPath) apuntando a `/mnt/data/postgres`.
- `postgres-pvc.yml` — PersistentVolumeClaim para la base de datos (solicitud de almacenamiento).
- `postgres-stateful.yml` — StatefulSet para desplegar PostgreSQL con inyección de secretos desde Vault (role `bd-vehicles`).
- `service.yml` — Service para exponer PostgreSQL en el puerto 5432.

---

## Requisitos previos

- Vault configurado y con secretos en `secret/data/ecomove/bd-vehicles`.
- El `ServiceAccount` `myapp-bd` debe existir (proporcionado por `k8s-vault/internal-bd.yml`).

---

## Orden de despliegue recomendado

1. Crear namespace `postgres`:

```bash
kubectl apply -f k8s-postgrest/ns.yml
```

2. Crear PV y PVC y desplegar el StatefulSet:

```bash
kubectl apply -f k8s-postgrest/posgrest-pv.yml
kubectl apply -f k8s-postgrest/postgres-pvc.yml
kubectl apply -f k8s-postgrest/postgres-stateful.yml
kubectl apply -f k8s-postgrest/service.yml
```

3. Verificar que PostgreSQL esté listando correctamente el storage y esté en estado 'Running'.

---

## Verificación y debug

```bash
kubectl get pods -n postgres
kubectl describe pod -l app=postgres -n postgres
kubectl get pvc -n postgres
kubectl logs -l app=postgres -n postgres
```

---

## Notas y consideraciones

- Como el PV usa `hostPath`, asegúrate de que el directorio exista en el host del nodo: `/mnt/data/postgres`.
- En entornos productivos, usar un StorageClass/CSI en vez de `hostPath`.
- Asegúrate de que los secretos de Vault sean correctos y que la inyección funcione para `POSTGRES_USER`, `POSTGRES_PASSWORD` y `POSTGRES_DB`.
