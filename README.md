# Evaluación Parcial N°3: Despliegue de Redmine + PostgreSQL en AWS EKS 
# Autor
Jesús Quijada 

## 1. Descripción de la Arquitectura
La arquitectura implementada despliega la herramienta de gestión de proyectos Redmine (frontend) conectada a una base de datos PostgreSQL (backend) sobre un cluster de Kubernetes en AWS EKS. Toda la infraestructura se encuentra alojada en el namespace `pmotrack`

El flujo de acceso permite a los usuarios interactuar con Redmine a través de internet mediante un Service de tipo NodePort. Por su parte, el backend está aislado y cuenta con persistencia de datos. Además, el frontend está protegido ante picos de tráfico mediante una política de autoescalado.

## 2. Decisiones Técnicas
Para el cumplimiento de los requerimientos de PMOTrack, se tomaron las siguientes decisiones de diseño:

* Imágenes de Contenedores: Se utilizaron las imágenes oficiales `postgres:16` para el motor de base de datos y `redmine:5` para la interfaz web.
* Gestión de Secretos: Para mantener la seguridad, las variables de entorno críticas (`POSTGRES_DB`, `POSTGRES_USER`, `POSTGRES_PASSWORD`) se extrajeron del manifiesto del Deployment y se aislaron utilizando un objeto `Secret` de Kubernetes.
* Tipos de Service: Para PostgreSQL, se implementó un Service de tipo `ClusterIP` en el puerto 5432. Esto restringe el acceso de la base de datos de forma estricta a la red interna del cluster, aumentando la seguridad.
* Para Redmine, se empleó un Service de tipo `NodePort` para exponer la aplicación hacia el exterior, permitiendo el acceso del usuario desde su navegador.
* Almacenamiento Persistente: Se configuró un `PersistentVolume` (PV) tipo `hostPath` junto con su respectivo `PersistentVolumeClaim` (PVC), montados en la ruta `/var/lib/postgresql/data` del contenedor de la base de datos. Esto asegura que la información persista en el disco del nodo y sobreviva al ciclo de vida efímero del Pod.
* Autoescalado (HPA): Se habilitó un `HorizontalPodAutoscaler` para el Deployment del frontend. Se estableció un límite mínimo de 1 réplica y un máximo de 5 réplicas, configurando un umbral de CPU del 50%. Esto garantiza disponibilidad y eficiencia de recursos bajo cargas imprevistas.

##3. Instrucciones de Despliegue 

Sigue estos pasos desde la terminal de AWS CloudShell para desplegar la arquitectura completa:

Paso 1: Preparación del cluster
```bash
 Ejecutar configuración inicial
bash commons/scripts/setup-cloudshell.sh

# Crear el cluster EKS
bash commons/scripts/create-cluster.sh 
# Crear el namespace
kubectl create namespace pmotrack

Paso 2: Despliegue del Backend (PostgreSQL)

Bash
# Aplicar manifiestos de base de datos, almacenamiento y secretos
kubectl apply -f manifests/postgres-secret.yaml -n pmotrack
kubectl apply -f manifests/postgres-storage.yaml -n pmotrack
kubectl apply -f manifests/postgres-deployment.yaml -n pmotrack
kubectl apply -f manifests/postgres-service.yaml -n pmotrack

Paso 3: Despliegue del Frontend Redmine

# Aplicar manifiestos de la aplicación web y su servicio de red
kubectl apply -f manifests/redmine-deployment.yaml -n pmotrack
kubectl apply -f manifests/redmine-service.yaml -n pmotrack

# Habilitar el acceso externo abriendo el NodePort asignado en el firewall de AWS
bash commons/scripts/open-nodeport.sh 31980

Paso 4: Configuración de autoescalado

# Instalar el servidor de métricas
kubectl apply -f [https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml](https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml)

# Aplicar manifiesto HPA
kubectl apply -f manifests/redmine-hpa.yaml -n pmotrack
