kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.3.0/standard-install.yaml
helm install ngf oci://ghcr.io/nginx/charts/nginx-gateway-fabric --create-namespace -n nginx-gateway
kubectl get pods,svc -n nginx-gateway
kubectl get gc

kubectl create deploy nginxgw --image=nginx --replicas=3
kubectl expose deploy nginxgw --port=80
kubectl apply -f http-routing.yaml
