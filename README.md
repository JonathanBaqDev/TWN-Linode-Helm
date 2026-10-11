# MongoDB on Linode using Helm
Reference project: https://gitlab.com/twn-devops-bootcamp/latest/10-kubernetes/helm-demo

## Provision a cluster on Linode
- Create a cluster with two nodes.
- Once it is provisioned, download the kubeconfig file, which contains the access credentials.
- Set the `KUBECONFIG` environment variable:
```bash
chmod 400 test-kubeconfig.yaml
export KUBECONFIG=test-kubeconfig.yaml

kubectl get nodes
```

## Install Helm
- Follow the installation instructions for your environment: https://helm.sh/docs/intro/install/

## MongoDB StatefulSets
- The MongoDB Helm chart is maintained by Bitnami: https://github.com/bitnami/charts/tree/main/bitnami/mongodb
- Create a values file to override settings such as the root password, persistence, and replica count. See [helm-mongodb.yaml](/helm-mongodb.yaml).
- Apply the values file when installing the chart:
```bash
helm install mongodb oci://registry-1.docker.io/bitnamicharts/mongodb -f helm-mongodb.yaml
```
- In the Linode console, check **Volumes**. Three persistent volumes should be provisioned, one for each replica pod.

## Mongo Express
- Create the Mongo Express Deployment and Service configuration. See [helm-mongo-express.yaml](/helm-mongo-express.yaml).
- To confirm the MongoDB password reference, inspect the Secret deployed in the previous step.
- Apply the configuration:
```bash
kubectl apply -f helm-mongo-express.yaml
kubectl get pods
kubectl logs <mongo-express-pod-name>
```

## Ingress
- Install the NGINX Ingress Controller Helm chart. See https://docs.nginx.com/nginx-ingress-controller/install/helm/open-source/.
```bash
helm install nginx-ingress oci://ghcr.io/nginx/charts/nginx-ingress --set controller.reportIngressStatus.enable=true --version 2.7.3
kubectl get ingressclasses
kubectl get pods
```
- `controller.reportIngressStatus.enable` updates the address field in Ingress resource status with the NGINX Ingress Controller's external address (public IP).
- This creates a NodeBalancer on Linode, which serves as the cluster's entry point.
- Create Ingress rules to direct traffic to the Mongo Express Service. You can set the host name to the Linode NodeBalancer's host name.
- See [helm-ingress.yaml](/helm-ingress.yaml):
```bash
kubectl apply -f helm-ingress.yaml
kubectl get ingress
```
- Open the NodeBalancer host name in a browser and confirm that you can access the Mongo Express UI.
- Make changes to the database, then verify persistence by removing and re-adding the pods. Your data should still be present afterward:
```bash
kubectl scale --replicas=0 statefulset/my-mongodb-release-mongodb
kubectl get pods
kubectl scale --replicas=3 statefulset/my-mongodb-release-mongodb
```
- Uninstall the MongoDB release with Helm:
```bash
helm ls
helm uninstall my-mongodb-release # This does not remove persistent volumes in Linode.
```
- Remove the volumes manually in Linode, or delete the entire cluster if needed.