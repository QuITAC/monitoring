# Monitoring
Monitoring stack using Grafana, Loki, Node Exporter and Prometheus to collect logs and system metrics.
View it under `grafana.queerreferat.ac`.

### Grafana 
Simple Grafana container to visualize metrics and logs.
Make sure to copy the grafana.ini.template and replace the username and password.
Datasource and Dashboard configurations are in their respective folder and should be created automatically on startup.
The two Datasources at the moment are Loki and Prometheus, which are explained below.

### Loki
Loki is a log aggregator from the same people who created Grafana.
In our case it works by installing the Docker plugin to use it as a log driver: https://grafana.com/docs/loki/latest/send-data/docker-driver/
Installation:
```bash
docker plugin install grafana/loki-docker-driver:2.9.4 --alias loki --grant-all-permissions
```

The `daemon.json` file is the Docker daemon config file which needs to be placed in `/etc/docker/daemon.json`.
It will set Loki as the default log driver such that each container automatically sends it's logs to Loki.
There are some additional settings which are necessary as Loki itself runs in a container,
so when it is shut down the Docker daemon runs into a deadlock as it cannot reach the log endpoint anymore.
Note, that the plugin does not run as a container, so Loki needs to be exposed to localhost to be reachable by Docker.

On each container you can set labels for Loki which can later be used in Grafana to query specific logs:
```yaml
logging:
  options:
    loki-external-labels: service=my-service
```

### Node Exporter
Node Exporter collects system metrics and exposes them on `http://127.0.0.1:9100/metrics`.
This endpoint is regularly scraped by Prometheus to collect these metrics.

Runs as a manually created systemd service named `node-expoter.service`.
The service file is in `/etc/systemd/system/node-exporter.service`.
It will run the binary in `/opt/node_exporter/node_exporter` and expose system metrics on port 9100 on localhost.
To view the logs run `sudo journalctl -e -u node-exporter`.

Installation:
```bash
# Download the latest binary from https://github.com/prometheus/node_exporter and place it into /opt/node_exporter/node_exporter
cp node-exporter.service /etc/systemd/system/node-exporter.system
sudo systemctl daemon-reload
sudo systemctl start node-exporter.service
sudo systemctl enable node-exporter.service
```

### Prometheus
Prometheus is a time-series database and will scrape it's targets in a given interval to collect metrics.
There are various exporters for all sorts of things, right now we're just using the node exporter for Linux system metrics.
Node exporter does not run in a container to avoid mounting the whole host file system, but as a systemd service instead.
This means that Prometheus needs to be able to access node exporter running on the host system loopback interface.
Therefore `host.docker.internal` is used and an additional rule in ufw is required to not block this traffic.
