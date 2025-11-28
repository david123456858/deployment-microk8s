# k8s-posgrest-images

📌 Descripción

Este directorio contiene los recursos de Kubernetes para la base de datos PostgreSQL que almacena datos relacionados con las imágenes (`postgres-images`). Incluye:

- `posgrest-pv.yml` — PersistentVolume (hostPath) apuntando a `/mnt/data/postgres`.
- `posgrest-pvc.yml` — PersistentVolumeClaim que solicita almacenamiento (500Mi o 5Gi según template).
- `posgrest-stateful.yml` — StatefulSet para desplegar PostgreSQL con inyección de secretos desde Vault (role `bd-images`).
- `service.yml` — Service para exponer PostgreSQL en el puerto 5432 dentro del namespace `postgres`.

---

## Requisitos previos

- Namespace `postgres` debe existir (ver `k8s-postgrest/ns.yml`).
- Vault debe estar configurado y contener los secretos para `bd-images` en `secret/data/ecomove/bd-images`.
- El `ServiceAccount` `myapp-bd` (de `k8s-vault/internal-bd.yml`) debe existir.

---

## Orden de despliegue recomendado

1. Crear el namespace `postgres` (si no existe):

```bash
kubectl apply -f k8s-postgrest/ns.yml
```

2. Crear PV, PVC y StatefulSet para postgres-images (en ese orden recomendado):

```bash
kubectl apply -f k8s-posgrest-images/posgrest-pv.yml
kubectl apply -f k8s-posgrest-images/posgrest-pvc.yml
kubectl apply -f k8s-posgrest-images/posgrest-stateful.yml
kubectl apply -f k8s-posgrest-images/service.yml
```

3. Verificar estado del StatefulSet y el PVC:

```bash
kubectl get statefulset -n postgres
kubectl get pvc -n postgres
kubectl get pv
```

---

## Verificación y debug

- Logs del pod postgres:

```bash
kubectl logs -l app=postgres-images -n postgres
```

- Describir el Pod para revisar si la inyección de secretos y mount de volúmenes se realizó correctamente:

```bash
kubectl describe pod -l app=postgres-images -n postgres
```

---

## Notas y consideraciones

- El PV usa `hostPath: /mnt/data/postgres`. En MicroK8s, asegúrate que esta ruta exista y sea accesible (permisos y espacio suficiente).
- Si planeas uso en producción, reemplaza `hostPath` por un backend de almacenamiento más robusto (NFS, CSI, etc.).
- Los secretos en Vault (por ejemplo, `DB_APP_USER_PASSWORD`) deben ser inyectados en el contenedor; revisa las políticas y role bindings en Vault.
