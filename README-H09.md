CST8918 - DevOps: Infrastructure as Code
Prof: Robert McKenney

# Hybrid-H09 Azure Kubernetes Service (AKS) Cluster with Terraform

## Background

As the Systems Engineer on your product development team, you have been tasked with creating an Azure Kubernetes Service (AKS) cluster using Terraform. The AKS cluster will be used to deploy a sample application that includes a frontend (using Vue), two backend services (one built with node, one with rust), and message broker (RabbitMQ).

Other team members have completed the application code and have provided you with a sample kubernetes configuration file that you will use to deploy the application to the AKS cluster.

## Instructions

### Use Terraform to create an Azure Kubernetes Service (AKS) cluster.

The AKS cluster should be created with the following configurations:

- a minimum of 1 nodes and a maximum of 3 nodes.
- created in a new resource group.
- use the latest Kubernetes version.
- use the "Standard_B2s" VM size
- use the "SystemAssigned" managed identity

> [!TIP]
> You will need to use the _azurerm_kubernetes_cluster_ resource to create the AKS cluster. Check the documentation to see the available configuration options.

### The output of the Terraform script should be the kubeconfig file for the AKS cluster.

```tf
output "kube_config" {
  value = azurerm_kubernetes_cluster.app.kube_config_raw

  sensitive = true
}
```

### Use the output of the Terraform script to connect to the AKS cluster using the `kubectl` command.

Save the output of the Terraform script to a file called `kubeconfig` and use the following command to connect to the AKS cluster:

```sh
echo "$(terraform output kube_config)" > ./kubeconfig
```

Check the file to make sure it contains the kubeconfig data:

```sh
cat ./kubeconfig
```

If the file begins with <<<eof and ends with eof, then you will need to edit the file and remove the <<<eof and eof lines.

Then use the following command to connect to the AKS cluster:

```sh
export KUBECONFIG=./kubeconfig
kubectl get nodes
```

If you are able to list the nodes in the AKS cluster, then you have successfully connected to the AKS cluster. If not, then you will need to review the previous steps and troubleshoot the issue.

### Deploy a sample application to the AKS cluster.

Use the provided `sample-app.yaml` file to deploy a sample application to the AKS cluster. This file contains the configuration for a multi-tier web application that includes a frontend (using Vue), two backend services (one built with node, one with rust), and message broker (RabbitMQ).

> [!NOTE]
> This sample application was borrowed from an Azure Quickstart template.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rabbitmq
spec:
  replicas: 1
  selector:
    matchLabels:
      app: rabbitmq
  template:
    metadata:
      labels:
        app: rabbitmq
    spec:
      nodeSelector:
        "kubernetes.io/os": linux
      containers:
        - name: rabbitmq
          image: mcr.microsoft.com/mirror/docker/library/rabbitmq:3.10-management-alpine
          ports:
            - containerPort: 5672
              name: rabbitmq-amqp
            - containerPort: 15672
              name: rabbitmq-http
          env:
            - name: RABBITMQ_DEFAULT_USER
              value: "username"
            - name: RABBITMQ_DEFAULT_PASS
              value: "password"
          resources:
            requests:
              cpu: 10m
              memory: 128Mi
            limits:
              cpu: 250m
              memory: 256Mi
          volumeMounts:
            - name: rabbitmq-enabled-plugins
              mountPath: /etc/rabbitmq/enabled_plugins
              subPath: enabled_plugins
      volumes:
        - name: rabbitmq-enabled-plugins
          configMap:
            name: rabbitmq-enabled-plugins
            items:
              - key: rabbitmq_enabled_plugins
                path: enabled_plugins
---
apiVersion: v1
data:
  rabbitmq_enabled_plugins: |
    [rabbitmq_management,rabbitmq_prometheus,rabbitmq_amqp1_0].
kind: ConfigMap
metadata:
  name: rabbitmq-enabled-plugins
---
apiVersion: v1
kind: Service
metadata:
  name: rabbitmq
spec:
  selector:
    app: rabbitmq
  ports:
    - name: rabbitmq-amqp
      port: 5672
      targetPort: 5672
    - name: rabbitmq-http
      port: 15672
      targetPort: 15672
  type: ClusterIP
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  replicas: 1
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
    spec:
      nodeSelector:
        "kubernetes.io/os": linux
      containers:
        - name: order-service
          image: ghcr.io/azure-samples/aks-store-demo/order-service:latest
          ports:
            - containerPort: 3000
          env:
            - name: ORDER_QUEUE_HOSTNAME
              value: "rabbitmq"
            - name: ORDER_QUEUE_PORT
              value: "5672"
            - name: ORDER_QUEUE_USERNAME
              value: "username"
            - name: ORDER_QUEUE_PASSWORD
              value: "password"
            - name: ORDER_QUEUE_NAME
              value: "orders"
            - name: FASTIFY_ADDRESS
              value: "0.0.0.0"
          resources:
            requests:
              cpu: 1m
              memory: 50Mi
            limits:
              cpu: 75m
              memory: 128Mi
      initContainers:
        - name: wait-for-rabbitmq
          image: busybox
          command:
            [
              "sh",
              "-c",
              "until nc -zv rabbitmq 5672; do echo waiting for rabbitmq; sleep 2; done;",
            ]
          resources:
            requests:
              cpu: 1m
              memory: 50Mi
            limits:
              cpu: 75m
              memory: 128Mi
---
apiVersion: v1
kind: Service
metadata:
  name: order-service
spec:
  type: ClusterIP
  ports:
    - name: http
      port: 3000
      targetPort: 3000
  selector:
    app: order-service
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: product-service
spec:
  replicas: 1
  selector:
    matchLabels:
      app: product-service
  template:
    metadata:
      labels:
        app: product-service
    spec:
      nodeSelector:
        "kubernetes.io/os": linux
      containers:
        - name: product-service
          image: ghcr.io/azure-samples/aks-store-demo/product-service:latest
          ports:
            - containerPort: 3002
          resources:
            requests:
              cpu: 1m
              memory: 1Mi
            limits:
              cpu: 1m
              memory: 7Mi
---
apiVersion: v1
kind: Service
metadata:
  name: product-service
spec:
  type: ClusterIP
  ports:
    - name: http
      port: 3002
      targetPort: 3002
  selector:
    app: product-service
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: store-front
spec:
  replicas: 1
  selector:
    matchLabels:
      app: store-front
  template:
    metadata:
      labels:
        app: store-front
    spec:
      nodeSelector:
        "kubernetes.io/os": linux
      containers:
        - name: store-front
          image: ghcr.io/azure-samples/aks-store-demo/store-front:latest
          ports:
            - containerPort: 8080
              name: store-front
          env:
            - name: VUE_APP_ORDER_SERVICE_URL
              value: "http://order-service:3000/"
            - name: VUE_APP_PRODUCT_SERVICE_URL
              value: "http://product-service:3002/"
          resources:
            requests:
              cpu: 1m
              memory: 200Mi
            limits:
              cpu: 1000m
              memory: 512Mi
---
apiVersion: v1
kind: Service
metadata:
  name: store-front
spec:
  ports:
    - port: 80
      targetPort: 8080
  selector:
    app: store-front
  type: LoadBalancer
```

#### Deploy the application

```bash
kubectl apply -f sample-app.yaml
```

You should see output similar to the following:

```plaintext
deployment.apps/rabbitmq created
service/rabbitmq created
deployment.apps/order-service created
service/order-service created
deployment.apps/product-service created
service/product-service created
deployment.apps/store-front created
service/store-front created
```

#### Verify the application is running

```bash
kubectl get pods
kubectl get services
```

Use the external IP address of the `store-front` service to access the application in a web browser.

## Submit

Your github repository for this assignment should be named `cst8918-w24-h09`.

When you have completed the assignment, submit the URL of your GitHub repository in the asignmnet folder on Brightspace.

## Clean up resources

When you're done with the application, you can clean up the resources you created in your Azure account to avoid incurring more charges.

You can use `kubectl` to delete the pods.

You can use `terraform` to destroy the infrastructure resources.
