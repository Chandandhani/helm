mkdir ~/helm-chart
cd ~/helm-chart
helm create ecommerce-mini
helm create release-pilot

mkdir -p ecommerce-mini/env
touch ecommerce-mini/env/dev-values.yaml
touch ecommerce-mini/env/uat-values.yaml
touch ecommerce-mini/env/prod-values.yaml

mkdir -p release-pilot/env
touch release-pilot/env/dev-values.yaml
touch release-pilot/env/uat-values.yaml
touch release-pilot/env/prod-values.yaml

tree .
vi ecommerce-mini/Chart.yaml
vi release-pilot/Chart.yaml

helm lint ecommerce-mini
helm lint release-pilot

helm template ecommerce-mini ecommerce-mini
helm template release-pilot release-pilot

helm package ecommerce-mini
helm package release-pilot

helm repo index . --url https://chandandhani.github.io/helm

helm repo add myrepo https://chandandhani.github.io/helm
curl -L -o dev-values.yaml https://raw.githubusercontent.com/Chandandhani/helm/main/release-pilot/env/dev-values.yaml

helm install ecommerce myrepo/ecommerce-mini -f dev-values.yaml
helm install pilot myrepo/release-pilot -f dev-values.yaml

helm upgrade ecommerce myrepo/ecommerce-mini

helm history ecommerce

helm rollback ecommerce 1
