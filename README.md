# Kubernetes practice with minikube

Independent learning exercises, not one application to install with `kubectl apply -f .`. Requires minikube and kubectl. Use the `practice` namespace; avoid mixing the standalone nginx Pod with the base Deployment, since both carry `app: nginx` and would be selected by the same Service.

## Deployment and service

```bash
minikube start
kubectl apply -f namespace.yaml
kubectl apply -n practice -f nginx-deployment.yaml -f nginx-service.yaml
kubectl rollout status -n practice deployment/nginx-deployment
kubectl port-forward -n practice service/nginx-service 8080:80
# In a second terminal: curl http://localhost:8080
```

The base Service is ClusterIP. Optional `nginx-service-dev.yaml` creates a separately named NodePort Service on 30081. Container probes check HTTP health and resource requests/limits are included.

## Ingress

```bash
minikube addons enable ingress
kubectl apply -n practice -f ingress.yaml
```

Route `shopflow.local` to the reachable ingress address (often `minikube ip` on native Linux). Docker Desktop/driver setups may need a tunnel; consult the minikube driver instructions. The Ingress references the base `nginx-service`.

## ConfigMap and Secret

```bash
kubectl apply -n practice -f nginx-configmap.yaml
cp app-secret.yaml.example app-secret.yaml
# Replace the dummy values for this local exercise; never commit real secrets.
kubectl apply -n practice -f app-secret.yaml -f nginx-deployment-with-secret.yaml
kubectl rollout status -n practice deployment/nginx-with-secret
```

These examples inject environment variables; the stock Nginx image does not use APP_NAME/DB_PASSWORD to configure an application. Kubernetes Secret encoding is not encryption. The ConfigMap-only Deployment and Pods are alternative exercises.

## Volumes

```bash
minikube ssh -- sudo mkdir -p /data/my-pv
kubectl apply -n practice -f persistant-volume.yaml -f pod-with-pvc.yaml
kubectl get pvc -n practice
kubectl wait -n practice --for=condition=Ready pod/pod-with-pvc --timeout=90s
kubectl exec -n practice pod-with-pvc -- sh -c 'echo hello > /data/probe.txt'
kubectl delete pod -n practice pod-with-pvc
kubectl apply -n practice -f pod-with-pvc.yaml
kubectl wait -n practice --for=condition=Ready pod/pod-with-pvc --timeout=90s
kubectl exec -n practice pod-with-pvc -- cat /data/probe.txt
```

The static PVC binds explicitly to `my-pv` using an empty StorageClass. `hostPath` is node-local and intended only for this single-node exercise. `emptyDir` in `pod-with-volume.yaml` survives a container restart but is removed with the Pod. A replacement Pod using the PVC should still read `hello`.

## Cleanup

```bash
kubectl delete namespace practice
kubectl delete pv my-pv
```

The PV defaults to Retain; the host directory may remain and can be inspected before manual removal. This is not a production storage design.
