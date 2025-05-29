# ElectricSQL on Datum

## Pre-requisites

### Datum Account

* If you don't already have a Datum account, you can sign up for one at <https://cloud.datum.net>

### Tools and Project Setup

If this is your first time using Datum, run through the following tutorials to set up your tools and create a project:

* Set Up Tools <https://docs.datum.net/docs/tasks/tools/>
  * We’ll need kubectl, datumctl, and an API key
* Create a Project <https://docs.datum.net/docs/tasks/create-project>
  * Make sure to get and switch to the kubectl context for your project.

### Set up GCP integration with Datum

* Follow the instructions in the GCP integration guide to set up your GCP
integration with Datum up through the "Create a Project" section. You do not
need to do the "Create a Workload" step.
  * <https://docs.datum.net/docs/tutorials/infra-provider-gcp/>
  * We'll be using the Location `my-gcp-us-south1-a` and Network `default` from the tutorial.

### What else you'll need

* Kubectl configured to use the context of your Datum project
* Postgres client for testing
* _Optional_: Grafana Cloud account for telemetry exporting

## Tutorial Walkthrough

### Clone this repository

* Clone this repository to your local machine

```shell
git clone https://github.com/datum-labs/ElectricSQL-on-Datum
cd ElectricSQL-on-Datum
```

### Deploy postgres

* Run `kubectl apply -f postgres/`
  * This will load all the files inside the postgres directory.
  * `postgres-init-sql.yaml` will create a configmap that will be used as the
  source of a postgres init script that will load some initial database contents.
  * The workload deployment is in `postgres.yaml`.
* Default password is “my-secret-password”
* Run `kubectl get workload -w` and wait for it to say `Available true`
* Get your postgres instance’s IP address
  * `kubectl get instance -o wide` and look for the ‘External IP’ column.

### Deploy Electric SQL backend and front end

* Edit the `electricsql/electricsql.yaml` file and replace “#MYPOSTGRESIP” with your postgres instance’s external IP address.
* Run `kubectl apply -f electricsql/` to deploy ElectricSQL and the frontend:
* Run `kubectl get workload -w` and wait for it to say `Available true`

### Deploy gateways

* Use `kubectl get instances -o wide` to locate the External IPs for ElectricSQL and Frontend instances.
* Update `gateway/endpoints.yaml` with External IPs for ElectricSQL and Frontend EndpointSlices. For example if your Electric SQL External IP is 1.2.3.4:

  ```yaml
  endpoints:
    - addresses:
        -  #ELECTRICSQLIP
  ```

  becomes

  ```yaml
  endpoints:
    - addresses:
        -  1.2.3.4
  ```

  Similarly, if your Electric SQL Frontend External IP is 5.6.7.8:

  ```yaml
  endpoints:
    - addresses:
        -  #ELECTRICSQLFRONTENDIP
  ```

  becomes

  ```yaml
  endpoints:
    - addresses:
        -  5.6.7.8
  ```

* Deploy the Datum Gateway using `kubectl apply -f gateway/`.

### Test your ElectricSQL Deployment

* Use a browser to navigate to the FQDN listed in `kubectl get gateways -o wide`
* The ElectricSQL demo React application should load, displaying the sample data from the scores table in JSON format.
* Use a PostgresSQL client to modify the data in the scores table, noting real time updates to the JSON structure in your browser window.

## Going Further

### Telemetry Exports to Grafana

* Create a Hosted Prometheus metrics token on Grafana Cloud
  * Connections -> Add new connection -> Hosted Prometheus metrics

* Choose “from my local prometheus server”, “send metrics from a single Prometheus instance” and “Directly”
* Name your token and click create. You’ll be showing something similar to the below:

```yaml
global:
scrape_interval: 60s
remote_write:
  - url: <https://prometheus-my-end-point.grafana.net/api/prom/push>
    basic_auth:
      username: 2448961
      password: glc_longrandomtoken
scrape_configs:
  - job_name: node
    static_configs:
      - targets: ["localhost:9090"]

```

* Make note of the username, password, and URL fields.
* Edit the `telemetry/grafana-cloud-credentials.yaml` file and replace the username and password with the values from Grafana Cloud. Use quotes around the username field since if it's just a number. It'll look something like this:

   ```yaml
  apiVersion: v1
  kind: Secret
  metadata:
    name: grafana-cloud-credentials
  type: kubernetes.io/basic-auth
  stringData:
    username: "12345678"
    password: "glc_longrandomtoken"
  ```
  
* Edit the `telemetry/exportpolicy.yaml` file and replace #GRAFANA_CLOUD_ENDPOINT with the URL from Grafana Cloud.

* Run `kubectl apply -f telemetry`
* Go to Drilldown->Metrics on Grafana Cloud and you should now see metrics prefixed with “datum_” showing up.
* Explore and see how your gateways are doing.
