# Lab 1 - CI/CD avec Jenkins

**Étudiante :** Meryem Elheni
**Cours :** Jenkins CI/CD Pipeline (Formateur : Wahid Hamdi)

## Objectif

Déployer une application Spring Boot dans Kubernetes (Minikube), d'abord manuellement, puis automatiquement avec Jenkins.

## Environnement

- Ubuntu 26.04 Server (VM VMware)
- Docker, Minikube, kubectl
- Java 17, Maven
- Jenkins 2.580.1

## Partie I : Déploiement manuel (sans Jenkins)

### 1. Compilation du projet

Le projet Spring Boot est compilé avec Maven. L'application écoute sur le port **2222** (`server.port=2222`).

```bash
mvn clean install
```

![Build Maven](01-mvn-build.png)

### 2. Création de l'image Docker

Dockerfile utilisé :

```dockerfile
FROM eclipse-temurin:17-jre
EXPOSE 2222
WORKDIR /app
COPY target/devOpsDemo-0.0.1-SNAPSHOT.jar app.jar
CMD ["java", "-jar", "app.jar"]
```

```bash
docker build -t devops-demo:1.0.0 .
```

![Docker images](02-docker-images.png)

### 3. Envoi sur Docker Hub

```bash
docker tag devops-demo:1.0.0 maryemelheni/devops-demo:1.0.0
docker push maryemelheni/devops-demo:1.0.0
```

![Docker push](03-docker-push.png)

### 4. Déploiement sur Kubernetes (Minikube)

`deployment.yaml` (2 réplicas) :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spring-app-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: spring-app
  template:
    metadata:
      labels:
        app: spring-app
    spec:
      containers:
        - name: spring-app-container
          image: maryemelheni/devops-demo:1.0.0
          ports:
            - containerPort: 2222
```

`service.yaml` (type NodePort) :

```yaml
apiVersion: v1
kind: Service
metadata:
  name: spring-app-service
spec:
  selector:
    app: spring-app
  ports:
    - protocol: TCP
      port: 2222
      targetPort: 2222
  type: NodePort
```

```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
```

![Pods](04-kubectl-pods.png)

![Service](05-kubectl-svc.png)

### 5. Test de l'application

```bash
curl http://$(minikube ip):30222
```

![Application](06-curl-app.png)

**Remarque :** le PDF du cours utilise le port 8080 dans les fichiers YAML, mais l'application écoute sur 2222. J'ai donc utilisé 2222 partout.

## Partie II : Jenkins Freestyle

Jenkins est accessible sur `http://192.168.65.138:8080`. Il tourne avec Java 21 (exigé par la version 2.580.1), alors que le projet utilise Java 17.

### Étape 1 : Job `freestyle_demo`

Création d'un projet free-style nommé `freestyle_demo` avec une étape de build « Exécuter un script shell » :

```bash
echo 'Bonjour'
ls -l
echo 'Bonjour DevOps' > test.txt
ls -l
```

Le fichier `test.txt` est créé dans `/var/lib/jenkins/workspace/freestyle_demo`.

### Étape 2 : Exécution de Test.java

Un fichier `Test.java` qui affiche « Hi Master DevOps » est placé dans le workspace, et la commande `java Test.java` est ajoutée au script.

```java
public class Test {
    public static void main(String[] args) {
        System.out.println("Hi Master DevOps");
    }
}
```

### Étape 3 : Script récupéré depuis GitHub

Le job est lié au dépôt `https://github.com/meryemelheni/jenkins-freestyle.git` (section **Gestion de code source**, branche `*/main`). Le dépôt contient `script.sh` et `Test.java`. L'étape de build devient :

```bash
sh script.sh
```

![Configuration du job](07-freestyle-config.png)

![Gestion de code source](09-git-config.png)

La console montre le clonage du dépôt, l'exécution du script et le résultat `Hi Master DevOps` :

![Console du build](08-freestyle-console.png)

### Étape 4 : Déclenchement automatique et rétention

- **Construire périodiquement** avec le planning `H/5 * * * *` : un build toutes les 5 minutes.
- **Supprimer les anciens builds** : seuls les 2 derniers builds sont conservés.
- **Supprimer le workspace avant le démarrage du build** : le dossier est vidé à chaque lancement.

![Rétention des builds](10a-retention.png)

![Trigger périodique](10b-trigger.png)

L'historique ne contient que 2 builds :

![Historique](11-historique-2-builds.png)
