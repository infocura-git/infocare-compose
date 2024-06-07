### Monitoring User
To be able to collect metrics from a database, a monitoring user should be defined with the right privileges.
A file containing all the grants is provided at _sql/monitor-grants.sql_.  
If you chose to name the monitoring user anything different then *infocare*, you will need to adapt this file accordingly.

### Network Traffic Rules
#### Grafana Cloud
To establish an SSH connection to Grafana Cloud, the PDC agent must run on a network that allows internet egress to the following endpoints: 

```
private-datasource-connect-prod-eu-west-2.grafana.net:22
private-datasource-connect-api-prod-eu-west-2.grafana.net:443
``` 

#### Dashboards
In order to access the local dashboards of the Infocare deplyment, the network must allow ingress traffic towards the infocare-container.


### PDC Token
In order for the Private Datasource Connect to run, a valid token must be supplied. This token needs to be generated when a new PDC network is defined in Grafana Cloud.


### Infocare Containers
Add a .env file with following contents
```shell
ICARE_PDC_TOKEN=<Token accuired through Infocura>
# For each monitored database, specify user credentials to be user in compose file
DB_USER="<database user>"
DB_PASS="<database pass>"
```

Define a file named ds.yml with folliwing contents
```yaml
apiVersion: 1
datasources:
- name: Prometheus
  type: prometheus
  orgId: 1
  url: http://prometheus:9090
  basicAuth: false
  isDefault: true
  version: 1
  editable: false
```

Define a file named config.river with following contents
```
logging {
        level = "warn"
}

prometheus.scrape "default" {
        scrape_interval = "10s"
        targets = concat(
                [{
                        job         = "db2metrics",
                        __address__ = "exporter1:9953",
                }],
		[{
                        job         = "db2metrics",
                        __address__ = "exporter2:9953",
                }],
        )

        forward_to = [
                prometheus.remote_write.local_prom.receiver,
        ]
}

prometheus.remote_write "local_prom" {
        endpoint {
                url = "http://prometheus:9090/api/v1/write"
        }
}
```
Users with SELinux enabled should run following command to set the correct context for the file
```
sudo chcon -Rt svirt_sandbox_file_t ./config.river
``` 

### Exporters

Define a file named "exporters.yml" with following content
```yaml
services:
  exporter_<dbname_1>:
    image: docker.io/infocura/infocare-db2-exporter
    ports:
      - "9953:9953"
    entrypoint:
      - /opt/run
      - "<db server ip or hostname>"
      - "<instance port>"
      - <database name>
      - ${DB_USER}
      - ${DB_PASS}
 
  exporter_<dbname_2>:
    image: docker.io/infocura/infocare-db2-exporter
    ports:
      - "9954:9953"
    entrypoint:
      - /opt/run
      - "<db server ip or hostname>"
      - "<instance port>"
      - <database name>
      - ${DB_USER}
      - ${DB_PASS}
```


Run with 
```shell
podman-compose --env-file ./.env -f docker-compose.yml -f exporters.yml up -d 
```
