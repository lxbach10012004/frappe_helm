# Hướng dẫn chạy với Helm Chart để deploy lên K8s
- Kết nối vào cluster (nếu chưa làm):
```
gcloud container clusters get-credentials erp-next-cluster   --region asia-southeast1   --project erpnext-ptuddn-454813
```
- Thực hiện cài đặt:
```
helm repo add frappe https://helm.erpnext.com
helm repo add nfs-ganesha-server-and-external-provisioner https://kubernetes-sigs.github.io/nfs-ganesha-server-and-external-provisioner
helm repo update
```
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.9.4/deploy/static/provider/cloud/deploy.yaml
```
Thêm cnay để lên đc HTTPS
```
helm repo add jetstack https://charts.jetstack.io --force-update
helm repo update

helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.17.2 \ # or your chosen version
  --set installCRDs=true

kubectl apply -f erpnext/cluster-issuer.yaml
```
```
kubectl create namespace nfs
helm repo add nfs-ganesha-server-and-external-provisioner https://kubernetes-sigs.github.io/nfs-ganesha-server-and-external-provisioner
helm upgrade --install -n nfs in-cluster nfs-ganesha-server-and-external-provisioner/nfs-server-provisioner \
  --set 'storageClass.mountOptions={vers=4.1}' \
  --set persistence.enabled=true \
  --set persistence.size=32Gi
```
- Đợi cái trên chạy xong mới chạy tiếp. Kiểm tra bằng ```kubectl get all -n nfs```. Thấy 1/1 hết là đc. (Thật ra ko đợi chắc cũng ko sao đâu, vì CI-CD chắc ko biết đợi) Rồi chạy tiếp cái sau:
```
kubectl create namespace erpnext
helm upgrade --install frappe-bench --namespace erpnext -f erpnext/values.yaml frappe/erpnext
```
- Kiểm tra bằng ```kubectl get all -n erpnext```. Thấy 2 cái pod "frappe-bench-erpnext-new-site" và "frappe-bench-erpnext-conf-bench" Completed và còn lại Running là xong.

- Truy cập vào ```http://app.erpnext-uet.great-site.net/```
- Tài khoản: "Administrator" - Mật khẩu: "admin"

## Cách gỡ cài đặt toàn bộ
```
helm uninstall frappe-bench -n erpnext
helm uninstall in-cluster -n nfs
```
```
kubectl delete pod --all --grace-period=0 --force -n nfs
kubectl delete pvc --all --grace-period=0 --force -n nfs
kubectl delete pv --all --grace-period=0 --force -n nfs
kubectl delete configmap --all --grace-period=0 --force -n nfs
kubectl delete secret --all --grace-period=0 --force -n nfs

kubectl delete pod --all --grace-period=0 --force -n erpnext
kubectl delete pvc --all --grace-period=0 --force -n erpnext
kubectl delete pv --all --grace-period=0 --force -n erpnext
kubectl delete configmap --all --grace-period=0 --force -n erpnext
kubectl delete secret --all --grace-period=0 --force -n erpnext

kubectl delete namespace erpnext
kubectl delete namespace nfs
```
# Contents

### Frappe/ERPNext Helm Chart

Helm Chart to deploy a *frappe-bench*-like environment on Kubernetes. It adds following resources:

ConfigMaps:

- `nginx-config` is used to override default.conf for nginx reverse proxy and static assets container.

Deployments:

- `gunicorn` deployment contains frappe/erpnext gunicorn.
- `nginx` deployment contains frappe/erpnext static assets and nginx reverse proxy.
- `scheduler` deployment contains frappe/erpnext scheduler.
- `socketio` deployment contains frappe/erpnext socketio.
- `worker-d` deployment contains frappe/erpnext default worker.
- `worker-l` deployment contains frappe/erpnext long worker.
- `worker-s` deployment contains frappe/erpnext short worker.

HorizontalPodAutoscalers:

- `gunicorn` hpa scales frappe/erpnext gunicorn deployment.
- `nginx` hpa scales frappe/erpnext nginx deployment.
- `socketio` hpa scales frappe/erpnext socketio deployment.
- `worker-d` hpa scales frappe/erpnext default worker deployment.
- `worker-l` hpa scales frappe/erpnext long worker deployment.
- `worker-s` hpa scales frappe/erpnext short worker deployment.

Ingresses:

- `ingress` with custom name can be dynamically generated using `helm template` and configured values.

Jobs:

- `vol-fix` job to fix volume permissions, changes the `uid` and `gid` to `1000:1000`.
- `bench-conf` job to configure db host, redis hosts and socketio port.
- `create-site` job to create new site.
- `drop-site` job to drop existing site.
- `backup-push` job to backup and optionally push backup to S3 for existing site.
- `migrate` job to migrate existing site.
- `custom` job to run custom additional commands and configuration.

PVC:

- `erpnext` persistent volume claim is used to allocate volume for sites and config deployed with this release
- `erpnext-logs` persistent volume claim is used to allocate volume for logs

Secrets:

- `secret` is used to store `db-root-password` for external db host

Services:

- `gunicorn` service exposes pods from gunicorn deployment.
- `nginx` service exposes pods from nginx deployment.
- `socketio` service exposes pods from socketio deployment.

ServiceAccounts:

- `erpnext` service account is used by all deployments.

### Release Wizard

This is a release script for maintainers. It does the following:

- Checks latest tag for given major release for frappe and erpnext using git.
- Validates that release always bumps up.
- Bumps values.yaml and Chart.yaml for release changes
- Adds git tag for chart version
- Push to selected remote

This will trigger workflow to publish new version of helm chart.
