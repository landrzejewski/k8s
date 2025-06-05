https://cloudnative-pg.io/documentation/1.16/quickstart/

sudo mkdir -p /mnt/data/postgres
sudo chown -R 777:777 /mnt/data/postgres  # UID 999 is default for postgres image

kubectl create secret generic cnpg-superuser-secret \
  --from-literal=username=postgres \
  --from-literal=password=yourpassword

helm repo add cnpg https://cloudnative-pg.github.io/charts
helm repo update
helm install cnpg cnpg/cloudnative-pg --namespace cnpg-system --create-namespace

kubectl get pods -n cnpg-system

kubectl get cluster cnpg-cluster
