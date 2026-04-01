# Azure Deployment
## Service apps
1. Create a Dockerfile

```
FROM eclipse-temurin:25-jre-noble

LABEL author="YOUR NAME" \
      description="SERVICE NAME AND DESCRIPTION" \
      version="1.0" \
      org.opencontainers.image.vendor="YOUR NAME" \
      org.opencontainers.image.title="SERVICE NAME"

EXPOSE 8080
EXPOSE 7980

RUN useradd -r -s /usr/sbin/nologin -U -d /opt/<service-name> appuser \
    && mkdir -p /opt/<service-name> \
    && chown -R appuser:appuser /opt/<service-name>

USER appuser
WORKDIR /opt/<service-name>

COPY --chown=appuser:appuser path/to/target/jar /opt/<service-name>/<jar-file>

ENTRYPOINT ["java", "--enable-native-access=ALL-UNNAMED", "-jar", "/opt/<service-name>/<jar-file>"]
```
  
2. Run Dockerfile and create tag

```
docker build -t my-api:1.0 . 

```

3. Create a resource group
```
az group create --name myResourceGroup --location eastus
```

4. Create AKS cluster
```
az aks create \
  --resource-group myResourceGroup \
  --name myAKSCluster \
  --node-count 2 \
  --enable-managed-identity \
  --generate-ssh-keys
```
5. Connect kubectl to your AKS cluster

```
az aks get-credentials --resource-group myResourceGroup --name myAKSCluster
kubectl get nodes
```

6. create deployment.yaml file

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-api
spec:
  replicas: 2
  selector:
    matchLabels:
      app: my-api
  template:
    metadata:
      labels:
        app: my-api
    spec:
      containers:
        - name: my-api
          image: docker.io/<your-dockerhub-username>/my-api:latest
          imagePullPolicy: Always
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "256Mi"
          readinessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 15
            periodSeconds: 20
```
Apply deployment file

```
kubectl apply -f deployment.yaml
```

7. create service.yaml (Ingress)
```
apiVersion: v1
kind: Service
metadata:
  name: my-api-service
spec:
  type: LoadBalancer
  selector:
    app: my-api
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
```
Apply service
```
kubectl apply -f service.yaml
```
Get service public IP
```
kubectl get svc
```

## AKV

1. create key vault
```
az keyvault create --name <kvName> --resource-group <rg>
```
2. add secret
```
az keyvault secret set \
  --vault-name <kvName> \
  --name MySecret \
  --value "super-secret-value"
```
3. enable secret store
```
az aks enable-addons \
  --addons azure-keyvault-secrets-provider \
  --name <aksName> \
  --resource-group <rg>
```
4. add AKS <-> KV permission
```
az keyvault set-policy \
  --name <kvName> \
  --secret-permissions get \
  --spn <aksManagedIdentityClientId>
```
5. create secretsproviderclass.yaml (manifest)
```
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: myapi-kv-secrets
spec:
  provider: azure
  parameters:
    usePodIdentity: "false"
    useVMManagedIdentity: "true"
    userAssignedIdentityID: "<managed-identity-client-id>"
    keyvaultName: "<kvName>"
    tenantId: "<tenant-id>"
    objects: |
      array:
        - |
          objectName: MySecret
          objectType: secret
          objectVersion: ""
  secretObjects:
    - secretName: myapi-secret
      type: Opaque
      data:
        - objectName: MySecret
          key: MySecret
```
Apply manifest
```
kubectl apply -f secretproviderclass.yaml
```
6. Add KV container to deployment.yaml file

```
spec:
  containers:
    - name: myapi
      image: <acrName>.azurecr.io/myapi:latest
      volumeMounts:
        - name: secrets-store
          mountPath: "/mnt/secrets"
          readOnly: true
      env:
        - name: MY_SECRET
          valueFrom:
            secretKeyRef:
              name: myapi-secret
              key: MySecret

  volumes:
    - name: secrets-store
      csi:
        driver: secrets-store.csi.k8s.io
        readOnly: true
        volumeAttributes:
          secretProviderClass: "myapi-kv-secrets"
```
7. Restart the deployment just to be sure
```
kubectl rollout restart deployment <deploymentName>
```

