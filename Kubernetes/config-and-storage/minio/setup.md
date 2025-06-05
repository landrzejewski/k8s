docker run -d -p 9000:9000 -p 9001:9001 \
--name minio \
-e "MINIO_ROOT_USER=minioadmin" \
-e "MINIO_ROOT_PASSWORD=minioadmin123" \
-v /mnt/data:/data \
quay.io/minio/minio server /data --console-address ":9001"


Access MinIO Console: http://localhost:9001
Create a bucket, e.g., my-bucket.


https://github.com/yandex-cloud/k8s-csi-s3

git clone https://github.com/yandex-cloud/csi-s3.git
cd csi-s3/deploy/kubernetes
kubectl apply -f .