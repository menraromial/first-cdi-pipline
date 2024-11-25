

### 1. Ajouter un Job Kubernetes pour les Migrations

Pour effectuer les migrations, vous pouvez créer un Job Kubernetes qui exécutera les commandes de migration avant de démarrer l'application Django.

#### a. Créer un fichier de configuration pour le Job de migration

Créez un fichier `django-migrate-job.yaml` :

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: django-migrate
spec:
  template:
    spec:
      containers:
      - name: django-migrate
        image: mydjangoapp:latest
        env:
        - name: DATABASE_URL
          value: "postgres://$(POSTGRES_USER):$(POSTGRES_PASSWORD)@postgres-service:5432/$(POSTGRES_DB)"
        envFrom:
        - secretRef:
            name: postgres-secret
        command: ["python", "manage.py", "migrate"]
      restartPolicy: OnFailure
```

#### b. Appliquer le Job de migration

Appliquez le Job de migration avant de déployer l'application Django :

```sh
kubectl apply -f django-migrate-job.yaml
```

### 2. Mettre à jour la procédure complète

Voici la procédure complète mise à jour avec l'étape de migration :

# Déploiement d'une Application Django avec PostgreSQL sur Kubernetes

Ce guide vous montre comment déployer une application Django utilisant PostgreSQL sur un cluster Kubernetes en utilisant des secrets pour les informations sensibles et des volumes pour la persistance des données.

## Prérequis

- Docker installé
- Kubernetes installé (minikube, kubeadm, etc.)
- kubectl configuré pour interagir avec votre cluster Kubernetes

## Étapes

### 1. Construction de l'image Docker

#### a. Créer un Dockerfile

Créez un fichier `Dockerfile` dans le répertoire racine de votre projet Django :

```Dockerfile
# Utiliser une image de base Python
FROM python:3.9-slim

# Définir les variables d'environnement
ENV PYTHONDONTWRITEBYTECODE 1
ENV PYTHONUNBUFFERED 1

# Installer les dépendances système
RUN apt-get update && apt-get install -y \
    build-essential \
    libpq-dev \
    && rm -rf /var/lib/apt/lists/*

# Définir le répertoire de travail
WORKDIR /app

# Copier les fichiers de l'application
COPY . /app

# Installer les dépendances Python
RUN pip install --no-cache-dir -r requirements.txt

# Exposer le port sur lequel l'application va tourner
EXPOSE 8000

# Commande pour démarrer l'application
CMD ["gunicorn", "--bind", "0.0.0.0:8000", "myproject.wsgi:application"]
```

#### b. Créer un fichier `requirements.txt`

Assurez-vous d'avoir un fichier `requirements.txt` dans votre répertoire de projet avec toutes les dépendances nécessaires :

```txt
Django>=3.2,<4.0
gunicorn
psycopg2-binary
```

#### c. Construire l'image Docker

Utilisez la commande suivante pour construire l'image Docker :

```sh
docker build -t mydjangoapp:latest .
```

### 2. Configuration de Kubernetes

#### a. Créer des Secrets pour les informations sensibles

Créez un fichier `secrets.yaml` pour stocker les informations sensibles comme les identifiants de la base de données :

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: postgres-secret
type: Opaque
data:
  POSTGRES_DB: bXlkYXRhYmFzZQ==  # Base64 encoded value of "mydatabase"
  POSTGRES_USER: dXNlcm5hbWU=    # Base64 encoded value of "username"
  POSTGRES_PASSWORD: cGFzc3dvcmQ= # Base64 encoded value of "password"
```

Pour encoder les valeurs en Base64, vous pouvez utiliser la commande suivante :

```sh
echo -n 'mydatabase' | base64
echo -n 'username' | base64
echo -n 'password' | base64
```

Appliquez les secrets :

```sh
kubectl apply -f secrets.yaml
```

#### b. Créer un PersistentVolumeClaim pour PostgreSQL

Créez un fichier `postgres-pvc.yaml` :

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

Appliquez le PersistentVolumeClaim :

```sh
kubectl apply -f postgres-pvc.yaml
```

#### c. Créer un fichier de configuration pour le déploiement de PostgreSQL

Créez un fichier `postgres-deployment.yaml` :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:13
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_DB
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: POSTGRES_DB
        - name: POSTGRES_USER
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: POSTGRES_USER
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: POSTGRES_PASSWORD
        volumeMounts:
        - mountPath: /var/lib/postgresql/data
          name: postgres-storage
      volumes:
      - name: postgres-storage
        persistentVolumeClaim:
          claimName: postgres-pvc
```

#### d. Créer un fichier de configuration pour le service PostgreSQL

Créez un fichier `postgres-service.yaml` :

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres-service
spec:
  selector:
    app: postgres
  ports:
    - protocol: TCP
      port: 5432
      targetPort: 5432
```

#### e. Créer un fichier de configuration pour le Job de migration

Créez un fichier `django-migrate-job.yaml` :

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: django-migrate
spec:
  template:
    spec:
      containers:
      - name: django-migrate
        image: mydjangoapp:latest
        env:
        - name: DATABASE_URL
          value: "postgres://$(POSTGRES_USER):$(POSTGRES_PASSWORD)@postgres-service:5432/$(POSTGRES_DB)"
        envFrom:
        - secretRef:
            name: postgres-secret
        command: ["python", "manage.py", "migrate"]
      restartPolicy: OnFailure
```

#### f. Créer un fichier de configuration pour le déploiement de Django

Créez un fichier `django-deployment.yaml` :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: django-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: django
  template:
    metadata:
      labels:
        app: django
    spec:
      containers:
      - name: django
        image: mydjangoapp:latest
        ports:
        - containerPort: 8000
        env:
        - name: DATABASE_URL
          value: "postgres://$(POSTGRES_USER):$(POSTGRES_PASSWORD)@postgres-service:5432/$(POSTGRES_DB)"
        envFrom:
        - secretRef:
            name: postgres-secret
```

#### g. Créer un fichier de configuration pour le service Django

Créez un fichier `django-service.yaml` :

```yaml
apiVersion: v1
kind: Service
metadata:
  name: django-service
spec:
  selector:
    app: django
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8000
  type: LoadBalancer
```

### 3. Déploiement dans Kubernetes

Appliquez les configurations :

```sh
kubectl apply -f postgres-deployment.yaml
kubectl apply -f postgres-service.yaml
kubectl apply -f django-migrate-job.yaml
kubectl apply -f django-deployment.yaml
kubectl apply -f django-service.yaml
```

### 4. Vérification

#### a. Vérifier les pods

Assurez-vous que les pods sont en cours d'exécution :

```sh
kubectl get pods
```

#### b. Vérifier les services

Assurez-vous que les services sont en cours d'exécution :

```sh
kubectl get services
```

### 5. Accéder à l'application

Si vous utilisez un service de type `LoadBalancer`, vous devriez obtenir une adresse IP externe pour accéder à votre application Django. Vous pouvez obtenir cette adresse IP en utilisant :

```sh
kubectl get services django-service
```

## Conclusion

Vous avez maintenant déployé votre application Django utilisant PostgreSQL dans un cluster Kubernetes avec des secrets pour les informations sensibles, des volumes pour la persistance des données, et un Job pour effectuer les migrations de la base de données. Cela améliore la sécurité et la fiabilité de votre déploiement.
