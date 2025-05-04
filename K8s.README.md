# Helm commands

🔗 **Web Helm**: [Helm Official Site](https://helm.sh/)

## Comandos básicos de Helm

* Crear configuración `helm create <nombre>`
* Aplicar configuración inicial: `helm install <nombre> .`
* Aplicar actualizaciones: `helm upgrade <nombre> .`

# K8s commands

* Obtener pods, deployments y services: `kubectl get <pods | deployments | services>`
* Revisar todos pods: `kubectl describe pods`
* Revisar un pod: `kubectl describe pod <nombre>`
* Eliminar pod: `kubectl delete pod <nombre>`
* Revisar logs: `kubectl logs <nombre>`

# Crear deployment:
```
kubectl create deployment <nombre> --image=<registro/url/imagen> --dry-run=client -o yaml > deployment.yml
```

# Crear service
```
kubectl create service clusterip <nombre> --tcp=<8888> --dry-run=client -o yaml > service.yml 
**kubectl create service nodeport <nombre> --tcp=<4000> --dry-run=client -o yaml > service.yml**
```
* **clusterip**: solo se puede acceder desde dentro del cluster
* **nodeport**: se puede acceder desde fuera del cluster

# Secrets

* Crear secretos, varios a la vez, o uno por uno.
```
kubectl create secret generic <nombre> --from-literal=key=value

kubectl create secret generic secret1 --from-literal=key1=value1 --from-literal=key2=value2
```
* Obtener los secretos `kubectl get secrets`
* Ver el contenido de un secreto `kubectl get secrets <nombre> -o yaml`

## Editar un secret
La forma más fácil es borrarlo y volverlo a crear pero si es más de un secret, no vamos a querer perder los demás.
Recordar que los secrets están en `base64`, por lo que si queremos editar un secret, debemos hacerlo en `base64`.

1. Editar el secret con `kubectl edit secret <nombre>` esto invocará el editor
2. Cambiar el valor (se puede usar un editor en [línea para convertir a base64](https://www.rapidtables.com/web/tools/base64-decode.html))
3. Tocar **i** para insertar líneas y editar el archivo
4. Poner el valor a decodificar en una nueva línea
5. Presionar **esc** y luego `:. ! base64 -D` para decodificar el valor
6. Presionar **i** para insertar o editar el valor
7. Presionar **esc** y luego `:. ! base64` para codificar el valor
8. Editar nuevamente el archivo **i** y dejar la línea en su posición
9. Presionar **esc** y luego **:wq** para guardar y salir

## Configurar secretos de Google Cloud para obtener las imágenes

1. Crear secreto:
```
kubectl create secret docker-registry gcr-json-key --docker-server=SERVIDOR-DE-GOOGLE-docker.pkg.dev --docker-username=_json_key --docker-password="$(cat 'PATH/DE/Tienda Microservices IAM.json')" --docker-email=TU_CORREO@gmail.com
```

2. Path del secreto para que use la llave:
```
kubectl patch serviceaccounts default -p '{ "imagePullSecrets": [{ "name":"gcr-json-key" }] }'
```

## Configurar secretos de AWS ECR para obtener las imágenes

### 1. Prerrequisitos
* Tener instalado y configurado el **AWS CLI** (`aws configure`) con un perfil que posea permisos para `ecr:GetAuthorizationToken`.
* Contar con un repositorio en ECR (y, opcionalmente, haber subido alguna imagen para probar).

### 2. Crear el secret `docker-registry`

> 🔒 Este secret almacena las credenciales generadas al vuelo por AWS para que Kubernetes pueda hacer **pull** de tus imágenes privadas.

```bash
# Sustituye: <aws_account_id>, <region>, <tu_correo@ejemplo.com>
# Crea (o actualiza) el secret llamado "ecr-registry"
aws ecr get-login-password --region <region> \
| kubectl create secret docker-registry ecr-registry \
    --docker-server=<aws_account_id>.dkr.ecr.<region>.amazonaws.com \
    --docker-username=AWS \
    --docker-password-stdin \
    --docker-email=<tu_correo@ejemplo.com> \
    --dry-run=client -o yaml \
| kubectl apply -f -
```

<details> <summary>¿Por qué <code>--docker-username=AWS</code>?</summary>
ECR usa el literal AWS como usuario y entrega un token (vía get-login-password) que funciona como contraseña.

</details>

### 3. Hacer que las cuentas de servicio usen el secret

Si quieres que todos los pods del namespace `default` (u otro) puedan tirar imágenes de ECR sin modificarlos uno a uno:

```bash
kubectl patch serviceaccount default \
  -p '{"imagePullSecrets":[{"name":"ecr-registry"}]}'
```

> Si ya había otros imagePullSecrets, añade este en la lista en lugar de sobrescribirla.

### 4. Referenciar el secret en manifiestos/Helm (opcional)

Si prefieres la vía explícita (por pod o chart), añade dentro del `Deployment` o en tu `values.yaml` de Helm:

```yaml
spec:
  template:
    spec:
      imagePullSecrets:
        - name: ecr-registry
```

### 🧪 Prueba rápida

1. **Build & push** una imagen a ECR:

```bash
aws ecr create-repository --repository-name demo --region <region>
docker build -t demo:latest .
docker tag demo:latest <aws_account_id>.dkr.ecr.<region>.amazonaws.com/demo:latest
aws ecr get-login-password --region <region> \
  | docker login --username AWS --password-stdin \
    <aws_account_id>.dkr.ecr.<region>.amazonaws.com
docker push <aws_account_id>.dkr.ecr.<region>.amazonaws.com/demo:latest
```

2. **Despliega** algo minimalista:

```bash
kubectl run ecr-test --image=<aws_account_id>.dkr.ecr.<region>.amazonaws.com/demo:latest --restart=Never
kubectl logs -f ecr-test
```
