# ssv-monitoring

A monitoring solution for node runners and validators utilizing docker containers with [Prometheus](https://prometheus.io/), [Grafana](http://grafana.org/), 
[NodeExporter](https://github.com/prometheus/node_exporter), and alerting with [AlertManager](https://github.com/prometheus/alertmanager). 

This is intended to be a single-stop solution for monitoring your signing state for SSV. The dashboard utilize the official
[SSV Monitoring dashboards](https://github.com/bloxapp/ssv/tree/main/monitoring/grafana)

## Install

Clone this repository on your Docker host, cd into ssv-monitoring directory and run compose up:

```bash
git clone https://github.com/LavenderFive/ssv-monitoring
cd ssv-monitoring

docker-compose up -d
```

**Caddy v2 does not accept plaintext passwords. It MUST be provided as a hash value. The above password hash corresponds to ADMIN_PASSWORD 'admin'. To know how to generate hash password, refer [Updating Caddy to v2](#Updating-Caddy-to-v2)**

Prerequisites:

* Docker Engine >= 1.13
* Docker Compose >= 1.11

Containers:

* Prometheus (metrics database) `http://<host-ip>:9090`
* Prometheus-Pushgateway (push acceptor for ephemeral and batch jobs) `http://<host-ip>:9091`
* AlertManager (alerts management) `http://<host-ip>:9093`
* Alertmanager-discord (disabled by default) `http://<host-ip>:9094`
* Grafana (visualize metrics) `http://<host-ip>:3000`
  * Infinity Plugin
* NodeExporter (host metrics collector)
* Caddy (reverse proxy and basic auth provider for prometheus and alertmanager)

## TL;DR: Steps
```
1. cp .env.sample .env
----- SSV -----
1. under prometheus/prometheus.yml line 43, add you SSV operators
1. under alertmanager/config.yml line 12, add your Pagerduty APIv2 service key
----- Caddy ------
1. under caddy/Caddyfile:
1. replace YOUR_WEBSITE.COM with your website
1. replace YOUR_EMAIL@EMAIL.COM with your email
1. point your dns to your monitoring server
-----------------
1. cd ~/ssv-monitoring
1. docker compose up -d
```

## Setup Grafana

### Grafana  Dashboard
This monitoring solution comes built in with a the official [SSV Monitoring dashboards](https://github.com/bloxapp/ssv/tree/main/monitoring/grafana), 
which works out of the box. Grafana, Prometheus, and Infinity are installed 
automatically.

---

Navigate to `http://<host-ip>:3000` and login with user ***admin*** password ***admin***. You can change the credentials in the compose file or by supplying the `ADMIN_USER` and `ADMIN_PASSWORD` environment variables on compose up. The config file can be added directly in grafana part like this

```yaml
grafana:
  image: grafana/grafana:7.2.0
  env_file:
    - .env
```

and the config file format should have this content

```yaml
GF_SECURITY_ADMIN_USER=admin
GF_SECURITY_ADMIN_PASSWORD=changeme
GF_USERS_ALLOW_SIGN_UP=false
```

If you want to change the password, you have to remove this entry, otherwise the change will not take effect

```yaml
- grafana_data:/var/lib/grafana
```

Grafana is preconfigured with dashboards and Prometheus as the default data source:

* Name: Prometheus
* Type: Prometheus
* Url: [http://prometheus:9090](http://prometheus:9090)
* Access: proxy

***SSV Node Dashboard***

![image4](https://github.com/LavenderFive/ssv-monitoring/assets/9121234/2ef7b653-0aec-457c-a676-eecc274c4cfe)


***Monitor Services Dashboard***

![Monitor Services](https://raw.githubusercontent.com/LavenderFive/ssv-monitoring/master/screens/Grafana_Prometheus.png)

The Monitor Services Dashboard shows key metrics for monitoring the containers that make up the monitoring stack:

* Prometheus container uptime, monitoring stack total memory usage, Prometheus local storage memory chunks and series
* Container CPU usage graph
* Container memory usage graph
* Prometheus chunks to persist and persistence urgency graphs
* Prometheus chunks ops and checkpoint duration graphs
* Prometheus samples ingested rate, target scrapes and scrape duration graphs
* Prometheus HTTP requests graph
* Prometheus alerts graph

## Define alerts

Two alert groups have been setup within the [alert.rules](https://github.com/LavenderFive/ssv-monitoring/blob/master/prometheus/alert.rules) configuration file:

* Monitoring services alerts [targets](https://github.com/LavenderFive/ssv-monitoring/blob/master/prometheus/alert.rules#L13-L22)
* Peggo alerts [peggo](https://github.com/LavenderFive/ssv-monitoring/blob/master/prometheus/alert.rules#L2-L11)

You can modify the alert rules and reload them by making a HTTP POST call to Prometheus:

```bash
curl -X POST http://admin:admin@<host-ip>:9090/-/reload
```

***Monitoring services alerts***

Trigger an alert if any of the monitoring targets (node-exporter and cAdvisor) are down for more than 30 seconds:

```yaml
- alert: monitor_service_down
    expr: up == 0
    for: 30s
    labels:
      severity: critical
    annotations:
      summary: "Monitor service non-operational"
      description: "Service {{ $labels.instance }} is down."
```

***SSV alerts***

Trigger an alert if SSV, the Beacon, or Execution nodes are down.

```yaml
- name: SSV
  rules:
  - alert: SSVBeaconNodeDown
    expr: ssv_beacon_status < 2
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "SSV <> ETH Beacon Node Down for instance: {{ $labels.instance }}"
  - alert: SSVExecutionNodeDown
    expr: ssv_eth1_status < 2
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "SSV <> ETH Execution Node Down for instance: {{ $labels.instance }}"
  - alert: SSVOperatorDown
    expr: (ssv_node_status + 1) < 2 or (absent(ssv_node_status) * 0) < 2
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "SSV Operator Down for instance: {{ $labels.instance }}"
  - alert: SSVLowPeers
    expr: ssv_p2p_all_connected_peers < 10
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "SSV Operator Peers low for instance: {{ $labels.instance }}"
  - alert: SSVLowPerformance
    expr: sum(rate(ssv_validator_roles_failed{instance=~"$instance.*"}[5m])) > sum(rate(ssv_validator_roles_submitted{instance=~"$instance.*"}[5m]))
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "SSV Failed Submissions too high for instance: {{ $labels.instance }}"
```


## Setup alerting

The AlertManager service is responsible for handling alerts sent by Prometheus server.
AlertManager can send notifications via email, Pushover, Slack, HipChat or any other system that exposes a webhook interface.
A complete list of integrations can be found [here](https://prometheus.io/docs/alerting/configuration).

You can view and silence notifications by accessing `http://<host-ip>:9093`.

The notification receivers can be configured in [alertmanager/config.yml](https://github.com/LavenderFive/ssv-monitoring/blob/master/alertmanager/config.yml) file.

To receive alerts via Slack you need to make a custom integration by choose ***incoming web hooks*** in your Slack team app page.
You can find more details on setting up Slack integration [here](http://www.robustperception.io/using-slack-with-the-alertmanager/).

Copy the Slack Webhook URL into the ***api_url*** field and specify a Slack ***channel***.

```yaml
route:
    receiver: 'slack'

receivers:
    - name: 'slack'
      slack_configs:
          - send_resolved: true
            text: "{{ .CommonAnnotations.description }}"
            username: 'Prometheus'
            channel: '#<channel>'
            api_url: 'https://hooks.slack.com/services/<webhook-id>'
```

![Slack Notifications](https://raw.githubusercontent.com/LavenderFive/ssv-monitoring/master/screens/Slack_Notifications.png)
