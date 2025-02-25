# etterlatte-gcp-migrering

Verktøy for å gjennomføre databasemigrering mellom Google Cloud SQL databaser på en sikker måte. 

Med denne fremgangsmåten er det også mulig å velge spesifikke tabeller som skal migreres. Dette i motsetning til andre verktøy som kun tilbyr 
migrering av hele databaser. 

## Forarbeid gcp-application

### 1. Start app for migrering

Start pod'en som inneholder verktøyene ved å applye til ditt namespace:
```
kubectl apply -f gcloud.yaml
```

### 2. Generer servicebruker

Hvis det allerede finnes en servicebruker kan denne benyttes. Hvis ikke kan det opprettes i GCP Console:

https://console.cloud.google.com/iam-admin/serviceaccounts/create?walkthrough_id=iam--create-service-account

### 3. Opprett nøkkel for servicebruker

- Gå til [IAM & Admin / Service accounts](https://console.cloud.google.com/iam-admin/serviceaccounts)
- Velg "actions" -> "Manage keys" -> "Add key"
- JSON-tokenet som opprettes må så legges i `secret.yaml`.


### 4. Apply secret

Når stegene over er utført kan du kjøre:

```shell
kubectl apply -f secret.yaml
```

### 5. Apply network policy

For at appen skal kunne kommunisere med andre apper sin database må network policy for pod-en legges til.

Ip'ene som refereres må peke på riktige public ip'er for databasene som skal aksesseres. Dette finner man i oversikten over 
databaseinstanser i GCP Console. 

Oppdater `network-<dev/prod>.yaml` med riktige ip'er og kjør: 

```shell
kubectl apply -f network-<dev/prod>.yaml
```

### 6. Restart pod for å få med siste endringer
Pod-en må startes på nytt for at den skal lese secret og network policy som ble lagt til.
```shell
kubectl scale deployment <POD_ID> --replicas=0
```
Vent til pod'en er slått av, kjør så:
```shell
kubectl scale deployment <POD_ID> --replicas=1
```


### 7. Logg inn og aktiver servicebruker

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

Instansen er nå klar til å koble seg på databaser og starte migreringen.

## Forarbeid databaser
Videre guide tar utgangspunkt i at skjemadefinisjoner (DDL) allerede er opprettet i 
databasen det migreres til. For å hente ut dette kan man kjøre:

```shell
pg_dump -h localhost -p 5432 -U <BRUKER> -d <DATABASE> --schema-only --exclude-table-data=flyway_schema_history
```

Det kan være lurt å tenke på om man vi vil legge på indeksene i etterkant da
insert med hele tabellen sammen med indeksering kan ta lang tid. 

For å kunne lese og skrive til databasene det skal migreres fra og til, må servicebrukeren få tilganger.
Dette gjøres på følgende måte:

### 1. Gi service-account brukeren tilgang til database-instansene
Service-account brukeren må legges inn manuelt i brukerseksjonen på database-instansene det skal migreres mellom i cloud console.
Velg "Add user account" og skriv inn hele brukernavnet (feks `migrering-user@etterlatte-prod-207c.iam.gserviceaccount.com`).

### 2. Sett opp les-tilgang i database det skal migreres fra
Før migrering må det gis tilgang til skjema og tabeller hvor det skal leses fra:
```postgresql
GRANT USAGE on SCHEMA public to "<SERVICEACCOUNT-USER>";
GRANT SELECT ON ALL TABLES IN SCHEMA public TO "<SERVICEACCOUNT-USER>";
```

### 3. Sett opp skriv-tilgang i database det skal migreres til
Videre må det gis tilgang til skjema og tabeller det skal skrives til:
```postgresql
GRANT USAGE on SCHEMA public to "<SERVICEACCOUNT-USER>";
GRANT INSERT ON ALL TABLES IN SCHEMA public TO "<SERVICEACCOUNT-USER>"; 
```

Alternativt rollen "cloudsqliamserviceaccount"


## Uføre migrering

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
Når data er dumpet til pod kan det gjenopprettes i ønsket database. Først må gjeldende `cloud_sql_proxy` fjernes, og ny proxy settes opp mot
databasen det skal importeres til som angitt i steg 1.

Videre kan import kjøres på følgende måte:
```shell
psql -h localhost -p 5432 -U <MIGRATION_USER> <DATABASE_NAME> -f /data/dump.sql
```

## Cleanup

Når du er ferdig med migrering kan du kjøre:

```shell
kubectl delete -f gcloud.yaml
kubectl delete -f secret.yaml
kubectl delete -f network-<dev/prod>.yaml
```


## Mulige problemer

### Feil ved dump av database
error: invalid command \N

Dette er ikke den reelle feilen, men kan være at dataene ikke er riktig eksportert eller at man mangler tilgang. 
Feilen kommer vanligvis helt i starten av kjøringen og kan være vanskelig å se. Eksempel
```shell
ERROR:  permission denied for table behandling_versjon
psql:/data/dump.sql:7662: error: invalid command \.`
```

Løsningen på dette er å gi `cloudsqliamserviceaccount` tilgang til tabellen det feiler for. Se _Forarbeid databaser (steg 2)_. 

### Feil ved oppkobling til database 
```shell
pg_dump: error: connection to server at "localhost" (127.0.0.1), port 5432 failed: FATAL:  password authentication failed for user "migration-user@etterlatte-prod-207c.iam"
```
Dette betyr vanligvis at service-account brukeren ikke er lagt til i database-instansen. Se _Forarbeid databaser (steg 1)_

### Ikke tilgang til nye tabeller, selv om det er gitt tilgang tidligere
Dersom nye tabeller er lagt til, må tilganger tildeles på nytt. Se _Forarbeid databaser (steg 2 og steg 3)_.


## Øvrig dokumentasjon på confluence
https://confluence.adeo.no/display/TE/Migreringssteg+for+database