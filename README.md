# tuova_on_prem_server

> ## IMPORTANT!
>
> This repository was created as part of the TUOVA Project (see https://tuova.savonia.fi and https://tuovaproject.github.io/). Some of the information presented here might assume familiarity with other repositories created during the course of the project. For more details on those, see the "Additional Information" section at the bottom of this README.
>
> ### Author
>
> Mika Leskinen \<mika.leskinen@savonia.fi\>\
> R&D Specialist\
> Savonia University of Applied Sciences
>
> [![DOI](https://zenodo.org/badge/1411358437.svg)](https://doi.org/10.5281/zenodo.23394511)
>
> **Everything is provided as-is, use at your own risk.**
>
> TUOVA Project co-funded by the European Union / Etelä-Savon ELY-keskus.
>
> ![Co-funded by the EU](co-fundedbythe-eu.png)

## General

This repository contains the server-side setup of a pilot system that was deployed during the course of the TUOVA project. The setup consists of the following:

- InfluxDB database for storing collected (unprocessed) machine data (e.g. electric current measurements)
- PostgreSQL database for storing processed/derived data (e.g. machine efficiency, profitability)
- Apache Airflow service for scheduling data processing tasks (e.g. getting raw data from InfluxDB and storing processed data to PostgeSQL)
- Nginx proxy server for managing external access to the above services (e.g. communication ports, TLS certificates).

The system is designed to be deployed on a physical (or virtual) server machine located in a private network, and while the system would work just as well within a cloud VM ("VPS"), additional configuration to the above services and/or the cloud environment is likely to be required.

## Prerequisites

- A physical (or virtual) Linux-based server machine
- The pilot deployment was confirmed to work well with the following resources:
    - 16 CPU cores
    - 16GB RAM
    - 100GB SSD
- (should work fine with way less than that)
- Some prior experience with certain Linux and Docker concepts might be required.

## Setup and deployment

### Environment variables

The environment configuration is defined in a `.env` file with the following variables (adjust as needed):

- `HOSTNAME`: hostname or IP of the server machine (should be accessible from the LAN)
- `POSTGRES_DB`: database name for PostgreSQL (for processed data) (e.g. `TUOVA_DB`)
- `POSTGRES_USER`: username for PostgreSQL (for processed data)
- `POSTGRES_PASSWORD`: password for PostgreSQL (for processed data)
- `AIRFLOW_POSTGRES_USER`: username for Airflow's internal PostgreSQL database (e.g. `airflow`)
- `AIRFLOW_POSTGRES_PASS`: password for Airflow's internal PostgreSQL database
- `_AIRFLOW_WWW_USER_USERNAME`: username for Airflow's Web UI (e.g. `YOUR_NAME`)
- `_AIRFLOW_WWW_USER_PASSWORD`: password for Airflow's Web UI
- `DOCKER_INFLUXDB_INIT_MODE`: just set this to `setup` (see https://docs.influxdata.com/influxdb/v2/install/use-docker-compose/ for more info)
- `DOCKER_INFLUXDB_INIT_USERNAME`: default username for InfluxDB (e.g. `admin`)
- `DOCKER_INFLUXDB_INIT_PASSWORD`: password for the InfluxDB default user
- `DOCKER_INFLUXDB_INIT_ORG`: default organization name for InfluxDB (e.g. `tuova-org`)
- `DOCKER_INFLUXDB_INIT_BUCKET`: default bucket name for InfluxDB (e.g. `tuova-bucket`)
- `DOCKER_INFLUXDB_INIT_ADMIN_TOKEN`: default token for the admin user (e.g. a long random string)
- `POSTGRES_PORT`: port for external access to PostgreSQL (for processed/derived data) (e.g. `30001`)
- `AIRFLOW_PORT`: port for external access to Airflow (e.g. `30003`)
- `INFLUXDB_PORT`: port for externall access to InfluxDB (e.g. `30004`).

### Initial setup

After setting the environment variables in `.env`, the deployment process would be as follows:

1. Install Docker on the server machine as per instructions stated here: https://docs.docker.com/engine/install/
2. Copy the following files onto the server:
    - `./docker-compose.yml`
    - `./nginx.conf`
    - `./.env`
3. Set up SSL/TLS certificates (or use existing ones):
    - create a new folder called `certs` into the same folder where `docker-compose.yml` is located (e.g. `mkdir ./certs`)
    - navigate into that folder and create a self-signed certificate and key with:
        - `openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -sha256 -days 3650 -nodes`
    - if using existing certificate files, just copy them to the `certs` folder as `cert.pem` and `key.pem`
        - check that the permissions of `key.pem` allow read/write for owner only (fix with e.g. `chmod 600 ./certs/key.pem` if needed)
4. Run `docker compose up -d` to start the services in daemon mode.

Note: depending on your container environment, it's possible that the same cert files can't be read by both `nginx` and `postgres` because of different UIDs/GIDs inside the containers. One way to resolve this is to create another folder in `certs` for postgres and copy the cert files there as follows:

```bash
mkdir ./certs/postgres
cp ./certs/*.pem ./certs/postgres
```

In this case you would also need to change the `volumes` part of the PostgreSQL service in `docker-compose.yml` by replacing `- ./certs:/certs:ro` with `- ./certs/postgres:/certs:ro`.

You might also need to adjust the permissions/ownership of all `.pem` files to the correct container UIDs/GIDs. For the images utilized here, the commands to do this would be as follows:

```bash
chown 101:101 ./certs/*.pem
chown 999:999 ./certs/postgres/*.pem
```

It is worth noting that setting UIDs and GIDs of files to something that might or might not exist on the host could be considered more of a workaround, and while not necessarily dangerous, it can cause problems later on (most notably confusion among sysadmins). Finally, it is also worth noting that SSL/TLS certificates are usually signed by trusted parties for a reason. Using self-signed certificates and disabling certificate validation with public-facing/production environments could be considered bad practice.

### PostgreSQL setup

Once the services defined in `docker-compose.yml` are running, the process of initializing the PostgreSQL database would be as follows:

1. Download the data model schema SQL file from https://raw.githubusercontent.com/TUOVAProject/tuotantotiedon-tietomalli/refs/heads/main/db_create.sql
2. Copy the downloaded SQL file to the server and into a folder that's mounted into the PostgreSQL container (e.g. `./data/postgres-data/db_create.sql`)
3. Get a shell into the PostgreSQL container with `docker exec -it tuova-etl-postgres bash`
4. Access the PostgreSQL database with `psql -U tuova TUOVA_DB`
5. In the `psql` shell, run `\i /var/lib/postgresql/db_create.sql` (executes the SQL statements in `db_create.db`)
6. Generate rows for the `dim_date` and `dim_time` tables
    - see the data model section in "Additional Information" below.

### InfluxDB setup

No further configuration is needed for the InfluxDB service. For general usage (e.g. creating buckets and access tokens), see the docs at https://docs.influxdata.com/influxdb/v2/. For utilizing InfluxDB in machine data collection, see the data collection and processing sections in "Additional Information" below.

### Airflow setup

No further configuration is needed for the Airflow service. For general usage (e.g. managing workflows), see https://airflow.apache.org/docs/. For utilizing Airflow in data processing pipelines, see the data processing sections in "Additional Information" below.

## Notes

If nothing in this README (or the code) makes sense, feel free to improve.

## LICENSE

See [LICENSE](LICENSE).

`./docker-compose.yml` is a modified version of the example Docker compose file referenced in Airflow docs at https://airflow.apache.org/docs/apache-airflow/stable/howto/docker-compose/index.html, and is licensed under Apache License 2.0. See [THIRD-PARTY-LICENSES](THIRD-PARTY-LICENSES) for terms and conditions.

## Additional Information

### The TUOVA Project

- https://tuova.savonia.fi
- https://tuovaproject.github.io/

### The data model ("Tuotantotiedon tietomalli")

- https://github.com/TUOVAProject/tuotantotiedon-tietomalli
- https://tuovaproject.github.io/tuotantotiedon-tietomalli/

### On-premises server environment

- https://github.com/TUOVAProject/tuova_on_prem_server

### Sensor data collection system

- https://github.com/TUOVAProject/tuova-iot

### Energy analyzer data collection system

- https://github.com/TUOVAProject/tuova_energymeter

### Sensor data ETL process

- https://github.com/TUOVAProject/tuova-sensor-etl

### Sensor data - further processing

- https://github.com/TUOVAProject/tuova_machinedata_processing
