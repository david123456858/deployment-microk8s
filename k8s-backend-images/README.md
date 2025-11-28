# k8s-backend-images

📌 Descripción

Este directorio contiene manifiestos para el microservicio que maneja las imágenes (ecomove-images). Contiene:

- `deployment.yml` — Deployment de la aplicación `ecomove-back-images` (imagen: `jdavidperalta/backend-pipeline-images:0.0.2`). Usa anotaciones para la inyección de secretos desde Vault para variables como `IMAGE_SERVICE_URL`, `JWT_SECRET`.
- `migration.yml` — Job para correr migraciones relacionadas con la DB de imágenes (drizzle). Usa la misma imagen que el deployment y la inyección de secretos.
- `service.yml` — Service tipo ClusterIP (puerto 3000).

---

## Requisitos previos

- Vault y sus secretos (en `secret/data/ecomove/images`) deben existir y ser accesibles.
- El `ServiceAccount` `myapp-sa` debe existir (desde `k8s-vault/internal-app.yml`).
- La base de datos de imágenes debe estar configurada; si la base de datos se provee desde `k8s-posgrest-images`, aplicar esos recursos primero.

---

## Orden de despliegue recomendado

1. Desplegar el `namespace` `development-ecomove` si aún no está creado:

```bash
kubectl apply -f k8s-backend/namespace.yml
```

2. Desplegar la base de datos de imágenes (`k8s-posgrest-images` y `k8s-postgrest` si corresponde):

```bash
kubectl apply -f k8s-posgrest-images/posgrest-pv.yml
kubectl apply -f k8s-posgrest-images/posgrest-pvc.yml
kubectl apply -f k8s-posgrest-images/posgrest-stateful.yml
kubectl apply -f k8s-posgrest-images/service.yml
```

3. Asegurar Vault y ServiceAccount (desde `k8s-vault`):

```bash
kubectl apply -f k8s-vault/internal-app.yml
kubectl apply -f k8s-vault/internal-bd.yml
kubectl apply -f k8s-vault/external-vault.yml
```

4. Ejecutar migraciones (si es necesario):

```bash
kubectl apply -f k8s-backend-images/migration.yml
kubectl logs job/drizzle-migrate-images -n development-ecomove
```

5. Desplegar el `deployment` y `service`:

```bash
kubectl apply -f k8s-backend-images/service.yml
kubectl apply -f k8s-backend-images/deployment.yml
```

---

## Verificación y debug

- Comprobar que los pods y el servicio estén arriba:

```bash
kubectl get pods -n development-ecomove -l app=ecomove-images
kubectl get svc -n development-ecomove ecomove-svc-images
```

- Ver logs de migración:

```bash
kubectl logs job/drizzle-migrate-images -n development-ecomove
```

---

## Notas

- El job de migración usa `npx drizzle-kit push`. Si tu pipeline de CI produce imágenes o artefactos build/ o alguna carpeta `build/drizzle.config.js`, revisa que exista en la imagen o ajústalo en el job.
- Las anotaciones de Vault requieren que el servicio de Vault esté accesible y configurado correctamente con roles y políticas adecuadas.
