# k8s-vault

📌 Descripción

Este directorio contiene manifiestos relacionados con la integración de Vault, y los ServiceAccounts que los pods usan para autenticación dentro del cluster.

- `external-vault.yml` — Servicio `external-vault` y Endpoints para enrutar a una instancia Vault que corre fuera del cluster (IP: `192.168.1.18`).
- `external-bd.yml` — Servicio `external-vault-bd` y Endpoints para la misma instancia en el namespace `postgres`.
- `internal-app.yml` — `ServiceAccount` `myapp-sa` para las aplicaciones (backend y frontend).
- `internal-bd.yml` — `ServiceAccount` `myapp-bd` para los servicios de base de datos.
- `service.yml` — Otro Service que se puede usar para enrutar a Vault (tipo ExternalName), asegurando la resolución desde el cluster.

---

## Requisitos previos

- Debe existir una instancia de Vault accesible en la IP configurada (en el repo: `192.168.1.18`) y la URL `http://external-vault.development-ecomove.svc.cluster.local:8200` debe resolver correctamente en el cluster (EndPoints y Services ya están creados aquí).
- Configurar roles, policies y secretos dentro de Vault para:
  - `secret/data/ecomove/vehicles` (backend)
  - `secret/data/ecomove/images` (images)
  - `secret/data/ecomove/bd-vehicles` y `secret/data/ecomove/bd-images` (database credentials)

---

## Orden de despliegue recomendado

1. Desplegar los ServiceAccounts y servicios del vault:

```bash
kubectl apply -f k8s-vault/internal-bd.yml
kubectl apply -f k8s-vault/internal-app.yml
kubectl apply -f k8s-vault/external-vault.yml
kubectl apply -f k8s-vault/external-bd.yml
kubectl apply -f k8s-vault/service.yml
```

2. Verificar que las cuentas de servicio estén creadas:

```bash
kubectl get sa -n development-ecomove
kubectl get sa -n postgres
```

---

## Verificación y debug

- Verificar que el servicio y endpoints apunten correctamente:

```bash
kubectl get svc -n development-ecomove external-vault
kubectl get endpoints -n development-ecomove external-vault
kubectl get svc -n postgres external-vault-bd
kubectl get endpoints -n postgres external-vault-bd
```

- Si Vault está fuera del cluster y necesita comunicar con el cluster, asegúrate que la IP especificada sea accesible y que la URL se resuelva desde el cluster. Revisa políticas de firewall si existen problemas.

---

## Notas y consideraciones

- `external-vault.yml` configura Endpoints con una IP específica (no es dinámico). Para producción, considera usar un LoadBalancer o el DNS gestionado.
- Verificar las políticas y roles en Vault para permitir que las aplicaciones y la base de datos obtengan sus secretos.
- Asegúrate de aplicar `internal-app.yml` y `internal-bd.yml` antes de desplegar los pods que dependen de `myapp-sa` y `myapp-bd`.
