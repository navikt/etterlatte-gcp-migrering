# etterlatte-gcp-migrering

Verktøy for å gjennomføre databasemigrering fra on-premises databaser til Google Cloud SQL.

## Forarbeid

### 1. Start app for migrering

Start pod'en som inneholder verktøyene ved å applye til ditt namespace:
```
kubectl apply -f gcloud.yaml
```

### 2. Generer servicebruker

OBS: Hvis det allerede finnes en servicebruker kan denne benyttes.

Hvis det _ikke_ finnes servicebruker kan det opprettes i GCP Console:

https://console.cloud.google.com/iam-admin/serviceaccounts/create?walkthrough_id=iam--create-service-account

### 3. Opprett nøkkel for servicebruker

- Gå til [IAM & Admin / Service accounts](https://console.cloud.google.com/iam-admin/serviceaccounts)
- Velg "actions" -> "Manage keys" -> "Add key"
- JSON-tokenet som opprettes må så legges i `secret.yaml`


### 4. Apply secrets

Når stegene over er utført kan du kjøre:

```shell
kubectl apply -f secret.yaml
```

_OBS: Pod-en må startes på nytt for at den skal lese nøkkelen som ble lagt til._


### 5. Apply network

For at appen skal kunne kommunisere med andre apper sin database må pod-en sin nettverkspolicy endres.

Hent ned gjeldende `network.yaml` og kjør: 

```shell
kubectl apply -f network.yaml
```


### 6. Aktiver servicebruker

Exec inn i pod'en:
```
kubectl exec -it <POD_ID> -- sh
```

Aktiver servicebruker: 

```shell
gcloud auth activate-service-account --key-file /var/run/secrets/nais.io/migration-user/user
```

Sett prosjekt (miljø) du skal migrere:

```shell
gcloud config set project <PROJECT_ID>
```

## Migrering

For at det skal være mulig for serviceaccount-brukeren å lese fra databasen som det skal gjøres dump av, må det gis tilganger
til skjemaet og til tabellene i denne databasen:

```postgresql
GRANT USAGE on SCHEMA public to "<SERVICEACCOUNT-USER>";
GRANT SELECT ON ALL TABLES IN SCHEMA public TO "<SERVICEACCOUNT-USER>";
```

### 1. Koble til proxy

Hent ut instansbeskrivelse fra gcloud:

```shell
gcloud sql instances describe <APP_NAME> --format="get(connectionName)" --project <PROJECT_ID>
```

Legg den til i dette kallet for å åpne proxy mot databasen:

```shell
cloud_sql_proxy -enable_iam_login -instances=<INSTANCE_NAME>=tcp:5432 &
```

OBS: Denne blir startet i bakgrunnen. For å avslutte den og gå mot annen instans/database må du drepe prosessen. 
Det kan gjøres ved å kjøre: 

```shell
ls -l /proc/*/exe
```

Kjør deretter `kill -9 <PID>`

_Gjeldende PID er tallet som står i stien til cloud_sql_proxy. Eks. `/proc/123/exe` betyr at PID er 123._

### 2. Dump data fra database til pod

```shell
pg_dump -h localhost -p 5432 -U <MIGRATION_USER> -d <DATABASE_NAME> -f /data/dump.sql --data-only --exclude-table-data=flyway_schema_history
```

### 3. Gjenopprett dumpet data

Når data er dumpet til pod kan det gjenopprettes i ønsket database. 

// TODO: Resten av bruksanvisningen...

https://confluence.adeo.no/display/TE/Migreringssteg+for+database

Dette burde samles på ett sted...

## Cleanup

Når du er ferdig med migrering kan du kjøre `cleanup_migration.sh` fra lokal maskin

```shell
bash cleanup_migration.sh
```
