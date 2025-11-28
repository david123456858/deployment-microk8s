# k8s-front-ecomove

📌 Descripción

Este directorio contiene los manifiestos para desplegar el frontend `ecomove-front` (imagen `keduardoquintero/eco-frontend:1.1`).

- `deployment.yml` — Deployment con anotaciones para inyección de secretos desde Vault (variables VITE_API_URL, VITE_IMAGE_API_URL).
- `service.yml` — Service tipo ClusterIP que expone el frontend en el puerto 80.
- `ingress.yml` — Ingress que mapea el dominio `ecomove.com` a la ruta `/` y hace rewrite a `ecomove-front-svc`.

---

## Requisitos previos

- Backend (`k8s-backend`) y el servicio de imágenes (`k8s-backend-images`) deben estar desplegados y funcionando para que el frontend apunte a endpoints correctos.
- Vault y `myapp-sa` deben estar configurados y accesibles (ver `k8s-vault`).
- Ingress Controller (e.g., nginx ingress) debe estar instalado y configurado en el cluster.

---

## Orden de despliegue recomendado

1. Asegurar que los microservicios backend (y las DBs) estén arriba:

```bash
kubectl apply -f k8s-postgrest/*
kubectl apply -f k8s-posgrest-images/*
kubectl apply -f k8s-backend/*
kubectl apply -f k8s-backend-images/*
```

2. Desplegar el frontend y su servicio:

```bash
kubectl apply -f k8s-front-ecomove/service.yml
kubectl apply -f k8s-front-ecomove/deployment.yml
```

3. Desplegar el Ingress para exponer el frontend (si usas dominio o IP):

```bash
kubectl apply -f k8s-front-ecomove/ingress.yml
```

---

## Verificación y debug

- Ver pods y servicios:

```bash
kubectl get pods -n development-ecomove -l app=ecomove-front
kubectl get svc -n development-ecomove ecomove-front-svc
kubectl describe ingress -n development-ecomove ecomove-ingress
```

- Revisar logs del contenedor frontend:

```bash
kubectl logs -l app=ecomove-front -n development-ecomove
```

---

## Notas

- Configure el `host` en `ingress.yml` o utilice la IP del cluster si no tienes dominio.
- Asegúrate de que las variables VITE del Vault apuntan a los servicios correctos (API y servicio de imágenes). Si tu frontend se sirve estáticamente y no requiere variables en tiempo de ejecución, es posible que no necesite Vault.
