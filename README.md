# Ververica

Repository containing manifests for
[ververica](https://www.ververica.com/getting-started)
installation. Never install the content of this repo on our clusters manually. This is all done by argo-cd.

## Dependencies

This chart pulls in `ververica` as a dependency. The version
used is specified in `Chart.yaml` in the `dependencies` section.
If you change the version in there, you need to then run

    $ helm dependency update

in order to have the chart downloaded to the `charts` directory
and then also commit that new version alongside with the altered
`Chart.yaml` file.

See the [Helm docs](https://helm.sh/docs/topics/charts/#chart-dependencies)
for details.


## Render helm charts locally

The following command renders the charts like argo-cd does to validate the content.

### local

```
 helm template --release-name ververica -n ververica --skip-tests \
  -a networking.istio.io/v1beta1 \
  -a forecastle.stakater.com/v1alpha1 \
  -a security.istio.io/v1beta1 \
  -a bitnami.com/v1alpha1 \
  -f values-local.yaml \
  --output-dir _render/local .
```

### dev

```
 helm template --release-name ververica -n ververica --skip-tests \
  -a networking.istio.io/v1beta1 \
  -a forecastle.stakater.com/v1alpha1 \
  -a security.istio.io/v1beta1 \
  -a bitnami.com/v1alpha1 \
  -f values-development.yaml \
  --output-dir _render/dev . 
```

### prod

```
 helm template --release-name ververica -n ververica --skip-tests \
  -a networking.istio.io/v1beta1 \
  -a forecastle.stakater.com/v1alpha1 \
  -a security.istio.io/v1beta1 \
  -a bitnami.com/v1alpha1 \
  -f values-production.yaml \
  --output-dir _render/prod . 
```

You can use this command to check if the output is as you expect. The `-a` parameters are needed since we use the
helm feature `.Capabilities.APIVersions.Has` to determine if a `CR` is installable in the cluster or not. Since
helm templating is designed to work offline we have to list the supported `CR`. Using `.Capabilities.APIVersions.Has`
feature in templating prevents sync errors in argo-cd if a `CR` can't be applied since its `CRD` isn't ready.

## Oracle CDC Demo
After getting the `oracle-db` namespace and deployment up, prepare the database for CDC connector.
Then follow the [instructions from ververica](https://github.com/Ververica-Steadforce/lab-cdc-oracle)
for DataStream Deployment (JAR).

### Preparing the Oracle DB
On the oracle-db namespace find the sys password for the database and use in the further steps
Run the following on the database pod:
```bash
sqlplus sys/<password>@//localhost:1521/ORCLCDB as sysdba
  ALTER DATABASE ADD SUPPLEMENTAL LOG DATA;
  CREATE TABLESPACE logminer_tbs DATAFILE '/opt/oracle/oradata/ORCLCDB/logminer_tbs.dbf' SIZE 25M REUSE AUTOEXTEND ON MAXSIZE UNLIMITED;
  exit
  
sqlplus sys/<password>@//localhost:1521/ORCLPDB1 as sysdba
  CREATE TABLESPACE logminer_tbs DATAFILE '/opt/oracle/oradata/ORCLCDB/ORCLPDB1/logminer_tbs.dbf' SIZE 25M REUSE AUTOEXTEND ON MAXSIZE UNLIMITED;
  exit

sqlplus sys/<password>@//localhost:1521/ORCLCDB as sysdba
  CREATE USER c##flinkuser IDENTIFIED BY flinkpw DEFAULT TABLESPACE logminer_tbs QUOTA UNLIMITED ON logminer_tbs CONTAINER=ALL;
  GRANT CREATE SESSION TO c##flinkuser CONTAINER=ALL;
  GRANT SET CONTAINER TO c##flinkuser CONTAINER=ALL;
  GRANT SELECT ON V_$DATABASE to c##flinkuser CONTAINER=ALL;
  GRANT FLASHBACK ANY TABLE TO c##flinkuser CONTAINER=ALL;
  GRANT SELECT ANY TABLE TO c##flinkuser CONTAINER=ALL;
  GRANT SELECT_CATALOG_ROLE TO c##flinkuser CONTAINER=ALL;
  GRANT EXECUTE_CATALOG_ROLE TO c##flinkuser CONTAINER=ALL;
  GRANT SELECT ANY TRANSACTION TO c##flinkuser CONTAINER=ALL;
  GRANT LOGMINING TO c##flinkuser CONTAINER=ALL;
  GRANT CREATE TABLE TO c##flinkuser CONTAINER=ALL;
  -- need not to execute if set scan.incremental.snapshot.enabled=true(default)
  GRANT LOCK ANY TABLE TO c##flinkuser CONTAINER=ALL;
  GRANT CREATE SEQUENCE TO c##flinkuser CONTAINER=ALL;

  GRANT EXECUTE ON DBMS_LOGMNR TO c##flinkuser CONTAINER=ALL;
  GRANT EXECUTE ON DBMS_LOGMNR_D TO c##flinkuser CONTAINER=ALL;

  GRANT SELECT ON V_$LOG TO c##flinkuser CONTAINER=ALL;
  GRANT SELECT ON V_$LOG_HISTORY TO c##flinkuser CONTAINER=ALL;
  GRANT SELECT ON V_$LOGMNR_LOGS TO c##flinkuser CONTAINER=ALL;
  GRANT SELECT ON V_$LOGMNR_CONTENTS TO c##flinkuser CONTAINER=ALL;
  GRANT SELECT ON V_$LOGMNR_PARAMETERS TO c##flinkuser CONTAINER=ALL;
  GRANT SELECT ON V_$LOGFILE TO c##flinkuser CONTAINER=ALL;
  GRANT SELECT ON V_$ARCHIVED_LOG TO c##flinkuser CONTAINER=ALL;
  GRANT SELECT ON V_$ARCHIVE_DEST_STATUS TO c##flinkuser CONTAINER=ALL;

  CREATE USER c##flinkuser2 IDENTIFIED BY flinkpw DEFAULT TABLESPACE logminer_tbs QUOTA UNLIMITED ON logminer_tbs CONTAINER=ALL;
  GRANT CREATE SESSION TO c##flinkuser2 CONTAINER=ALL;
  GRANT SET CONTAINER TO c##flinkuser2 CONTAINER=ALL;
  GRANT SELECT ON V_$DATABASE to c##flinkuser2 CONTAINER=ALL;
  GRANT FLASHBACK ANY TABLE TO c##flinkuser2 CONTAINER=ALL;
  GRANT SELECT ANY TABLE TO c##flinkuser2 CONTAINER=ALL;
  GRANT SELECT_CATALOG_ROLE TO c##flinkuser2 CONTAINER=ALL;
  GRANT EXECUTE_CATALOG_ROLE TO c##flinkuser2 CONTAINER=ALL;
  GRANT SELECT ANY TRANSACTION TO c##flinkuser2 CONTAINER=ALL;
  GRANT LOGMINING TO c##flinkuser2 CONTAINER=ALL;
  GRANT CREATE TABLE TO c##flinkuser2 CONTAINER=ALL;
  -- need not to execute if set scan.incremental.snapshot.enabled=true(default)
  GRANT LOCK ANY TABLE TO c##flinkuser2 CONTAINER=ALL;
  GRANT CREATE SEQUENCE TO c##flinkuser2 CONTAINER=ALL;

  GRANT EXECUTE ON DBMS_LOGMNR TO c##flinkuser2 CONTAINER=ALL;
  GRANT EXECUTE ON DBMS_LOGMNR_D TO c##flinkuser2 CONTAINER=ALL;

  GRANT SELECT ON V_$LOG TO c##flinkuser2 CONTAINER=ALL;
  GRANT SELECT ON V_$LOG_HISTORY TO c##flinkuser2 CONTAINER=ALL;
  GRANT SELECT ON V_$LOGMNR_LOGS TO c##flinkuser2 CONTAINER=ALL;
  GRANT SELECT ON V_$LOGMNR_CONTENTS TO c##flinkuser2 CONTAINER=ALL;
  GRANT SELECT ON V_$LOGMNR_PARAMETERS TO c##flinkuser2 CONTAINER=ALL;
  GRANT SELECT ON V_$LOGFILE TO c##flinkuser2 CONTAINER=ALL;
  GRANT SELECT ON V_$ARCHIVED_LOG TO c##flinkuser2 CONTAINER=ALL;
  GRANT SELECT ON V_$ARCHIVE_DEST_STATUS TO c##flinkuser2 CONTAINER=ALL;
  exit

```

### Deploying the DataStream Deployment from Ververica
Note the following before following the instructions:
* the namespace in the example is `oracle` while we use `oracle-db`. make sure to correct that in 
  [application.conf](https://github.com/Ververica-Steadforce/lab-cdc-oracle/blob/main/src/main/resources/application.conf)
  before assembling the JAR
* you may limit the list of streamed tables by defining them in sourceTableList in `application.conf`
Then follow the instructions here: https://github.com/Ververica-Steadforce/lab-cdc-oracle


### Running the show-case
1. ensure you're connected to `ORCLCDB` via `flinkuser`.
2. follow [these](https://github.com/Ververica-Steadforce/lab-cdc-oracle/tree/main?tab=readme-ov-file#show-cases) steps.
