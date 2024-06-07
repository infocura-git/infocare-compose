### Before we begin

Make sure a monitor user exists on the database server and grant that user the right privileges


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
