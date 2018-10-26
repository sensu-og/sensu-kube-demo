# Sensu Kubernetes Monitoring Demo

## Prerequisites

1. __[Install Docker for Mac (Edge)](https://store.docker.com/editions/community/docker-ce-desktop-mac)__

2. __Enable Kubernetes (in the Docker for Mac preferences)__

<img src="https://github.com/sensu/sensu-kube-demo/raw/master/images/docker-kubernetes.png" width="600">

3. __Deploy the [Kubernetes NGINX Ingress Controller](https://github.com/kubernetes/ingress-nginx)__

   ```
   $ kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/mandatory.yaml
   ```

   Then use the modified "ingress-nginx" Kubernetes Service definition (works with Docker for Mac):

   ```
   $ kubectl create -f classic/ingress-nginx/services/ingress-nginx.yaml
   ```

4. __Add hostnames to /etc/hosts__

   ```
   $ sudo vi /etc/hosts

   127.0.0.1       sensu.local webui.sensu.local sensu-enterprise.local dashboard.sensu-enterprise.local
   127.0.0.1       influxdb.local grafana.local dummy.local
   ```

5. __Create Kubernetes Ingress Resources__

   ```
   $ kubectl create -f classic/ingress-nginx/ingress/sensu-enterprise.yaml
   $ kubectl create -f go/ingress-nginx/ingress/sensu-go.yaml
   ```

## Sensu Go Demo

### Deploy Application

1. Deploy dummy app pods

   ```
   $ kubectl apply -f go/deploy/dummy.yaml

   $ kubectl get pods

   $ curl -i http://dummy.local
   ```

### Sensu Backend

1. Deploy Sensu Backend

   ```
   $ kubectl create -f go/deploy/sensu-backend.yaml

   $ kubectl get pods
   ```

2. Configure `sensuctl` to use the built-in "admin" user

   ```
   $ sensuctl configure
   ```

### Multitenancy

1. Create "acme" organization

   ```
   $ sensuctl organization create acme

   $ sensuctl config set-organization acme
   ```

2. Create "demo" environment within the "acme" organization

   ```
   $ sensuctl environment create demo --interactive

   $ sensuctl environment list

   $ sensuctl config set-environment demo
   ```

3. Create "dev" user role with full-access to the "demo" environment

   ```
   $ sensuctl role create dev -t '*' \
   --create --delete --update --read \
   --environment demo --organization acme
   ```

4. Create "demo" user with the "dev" role

   ```
   $ sensuctl user create demo --interactive
   ```

5. Reconfigure `sensuctl` to use the "demo" user, "acme" organization", and "demo" environment

   ```
   $ sensuctl configure
   ```

### Deploy InfluxDB

1. Create a Kubernetes ConfigMap for InfluxDB configuration

   ```
   $ kubectl create configmap influxdb-config --from-file go/configmaps/influxdb.conf
   ```

2. Deploy InfluxDB with a Sensu Agent sidecar

    ```
    $ kubectl create -f go/deploy/influxdb.sensu.yaml

    $ kubectl get pods

    $ sensuctl entity list
    ```

### Sensu InfluxDB Event Handler

1. Create "influxdb" event handler for sending Sensu 2.0 metrics to InfluxDB

   ```
   $ cat go/config/handlers/influxdb.json

   $ sensuctl create -f go/config/handlers/influxdb.json

   $ sensuctl handler info influxdb
   ```

### Deploy Application

1. Deploy dummy app Sensu Agent sidecars

   ```
   $ kubectl apply -f go/deploy/dummy.sensu.yaml

   $ kubectl get pods

   $ curl -i http://dummy.local
   ```

### Sensu Monitoring Checks

1. Register a Sensu 2.0 Asset for check plugins

   ```
   $ cat go/config/assets/check-plugins.json

   $ sensuctl create -f go/config/assets/check-plugins.json

   $ sensuctl asset info check-plugins
   ```

2. Create a check to monitor dummy app /healthz

   ```
   $ sensuctl create -f go/config/checks/dummy-app-healthz.json

   $ sensuctl check info dummy-app-healthz

   $ sensuctl event list
   ```

3. Toggle the dummy app /healthz status

   ```
   $ curl -iXPOST http://dummy.local/healthz

   $ sensuctl event list
   ```

### Prometheus Scraping

1. Register a Sensu 2.0 Asset for the Prometheus metric collector

   ```
   $ sensuctl create -f go/config/assets/prometheus-collector.json
   ```

2. Create a check to collect dummy app Prometheus metrics

   ```
   $ sensuctl create -f go/config/checks/dummy-app-prometheus.json

   $ sensuctl check info dummy-app-prometheus
   ```

3. Query InfluxDB to list the stored series

   ```
   $ curl -GET 'http://influxdb.local/query' --data-urlencode 'q=SHOW SERIES ON sensu'
   ```

### Deploy Grafana

1. Deploy Grafana with a Sensu Agent sidecar

    ```
    $ kubectl create -f go/deploy/grafana.sensu.yaml

    $ kubectl get pods

    $ sensuctl entity list
    ```

### Grafana Data Source

In the Grafana WebUI (http://grafana.local), add the [InfluxDB data source](http://docs.grafana.org/features/datasources/influxdb/).

| Setting | Value |
| --- | --- |
| Type | InfluxDB |
| URL | http://influxdb.default.svc.cluster.local:8086 |
| Database | sensu |
| User | sensu |
| Password | password |