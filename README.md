# ciphertrust-manager-prometheus-grafana-example

Example for how to bring up [Prometheus](https://prometheus.io/) and [Grafana](https://grafana.com/)
with the purpose of visualizing metrics such as
CPU usage and memory usage of [CipherTrust Manager](https://thalesdocs.com/ctp/cm/latest/get_started/deployment/index.html) (CM).


## Pre-Requirements

* CipherTrust Manager v2.7.0 or later
* Docker
* Docker Compose (`docker-compose`)
* GNU Make


## Authentication

Enable metrics for example using the CM API Playground (`POST to /v1/system/metrics/prometheus/enable`).

Or use the CM CLI tool:

    ksctl metrics prometheus enable

The API token returned can be used for authenticating with the CM to get metrics.

Metrics can then be retrieved using the CM CLI, for example:

    ksctl metrics prometheus get --api-token <api-token>

This API token need to be passed to the Prometheus Docker container by updating
the `prometheus.yml` file.


## Configuration

Pre-required configuration changes in `prometheus.yml`:

 - Update CipherTrust Manager IP addresses, see `targets`. For a cluster it is
   possible to add several targets (IP addresses).
 - Update CipherTrust Manager Prometheus Metrics API Token, see `bearer_token`.

It is possible to have multiple scrape targets which might or might not share
the same Prometheus API Token.

Here is an example with 3 nodes of which 2 shares the same Prometheus API Token.

```
scrape_configs:
  - job_name: "CipherTrust Manager"
    scheme: "https"
    tls_config:
      #ca_file: "/trusted_cas/web-keysecure-local.pem"
      #server_name: "web.keysecure.local"
      insecure_skip_verify: true
    bearer_token: "1zplR4njZsRN5dNeWAFXhkL1x7MU9q4H"
    metrics_path: "/api/v1/system/metrics/prometheus"
    static_configs:
      - targets:
        - "54.80.181.243"
        - "3.93.47.9"
  - job_name: "CipherTrust Manager Staging"
    scheme: "https"
    tls_config:
      #ca_file: "/trusted_cas/web-keysecure-local.pem"
      #server_name: "web.keysecure.local"
      insecure_skip_verify: true
    bearer_token: "TnRHpdL9v8MnWv8DhN9xuAaKgPevMEZs"
    metrics_path: "/api/v1/system/metrics/prometheus"
    static_configs:
      - targets:
        - "1.2.3.4"
```


### SSL/TLS Setup

By default, the Prometheus configuration sets `insecure_skip_verify: true`
which is not recommended for production deployments as it skips SSL/TLS
certificate validation for the CipherTrust Manager server.

To trust a CA, download and store the CA certificate into the folder
`trusted_cas`. Then configure the path and the server name of the certificate
in the Prometheus configuration file (`prometheus.yml`), for example:

```
scrape_configs:
  - job_name: "CipherTrust Manager"
    scheme: "https"
    tls_config:
      ca_file: "/trusted_cas/web-keysecure-local.pem"
      server_name: "web.keysecure.local"
      #insecure_skip_verify: true
    bearer_token: "TnRHpdL9v8MnWv8DhN9xuAaKgPevMEZs"
    metrics_path: "/api/v1/system/metrics/prometheus"
    static_configs:
      - targets:
        - "1.2.3.4"
```


## Docker Containers

The Prometheus and Grafana Docker containers have their parameters specified in
the `docker-compose.yml` file.

To start the stack:

    make up

To stop the stack:

    make down

To stop the stack and clear all persisted data:

    make clear


## Prometheus (the database)

The file `prometheus.yml` contains Prometheus specific parameters such as the
target's (CM) IP address, etc. The IP address is just an example and need to be
modified.

Open Prometheus http://localhost:9090.

The state of the target can be determined from the `Status|Target` menu. It
should indicate that the state should be `UP`, and there should be no error.

If either of these are wrong, then verify the metrics endpoint is working by doing a sanity test via the CLI tool (`ksctl metrics prometheus get --api-token <api-token>`), or curl (`curl -k 'https://<host-name>/api/v1/system/metrics/prometheus' -H 'Authorization: Bearer <api-token>' --compressed
`). The Docker Compose logs can be used to debug problems too (`docker-compose logs -f`).


### Retention

By default Prometheus keeps a history of 30 days or 1 GB, whichever limit is
reached first will be enforced. These values can be changed in
`docker-compose.yml`, see the command line parameters
`storage.tsdb.retention.time` and `storage.tsdb.retention.size` respectively.


## Grafana (the UI)

Open Grafana http://localhost:3000.

1. Login

    * User: admin
    * Password: admin

2. Go to Dashboards -> Home and you should see a list of the included dashboards


### Dashboards

By default this stack comes bundled with four dashboards:

- CipherTrust Manager Services

  This dashboards shows statistics about resources specific to CipherTrust
  Manager such as the number of audit records and keys and the rate of creation
  for them. It also for example displays the number of crypto operations.

- CipherTrust Manager HTTP Traffic

  This dashbaord shows statistics about the HTTP API such as the numbers of
  requests per second per API and the average response time. It also for example
  shows the number of HTTP 500 errors.

- CipherTrust Manager Services

  This dashbaord shows statistics about the various services running inside
  the appliance with information such as the CPU usage and memory usage.

- CipherTrust Manager Developer

  This dashbaord shows statistics various statistics more targeted for
  developers with deep knowledge about the inner details of the product.
