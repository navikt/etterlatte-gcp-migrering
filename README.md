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

# Før migrering må man grante usage på schema og på tabeller
```GRANT USAGE on SCHEMA public to "cloudsqliamserviceaccount";```

```GRANT INSERT ON ALL TABLES IN SCHEMA public TO "cloudsqliamserviceaccount"; ```

#### OBS! Er nye tabeller lagt til må de også få spesifikk GRANT kjørt på seg. 
Dette må da gjøres via deploy(anbefalt) eller via superbruker da utviklere via iam tilgang
ikke har lov til å grante via ` nais postgres proxy`.
Eksempel: 
`kubectl get secret google-sql-etterlatte-vilkaarsvurdering -o json | jq '.data | map_values(@base64d)'`
og logg inn som rot.(anbefales ikke)

## Legg inn tabeller uten flyway tabeller i app man ønsker å migrere til
Her er det veldig lurt å tenke på om man vi vil legge på indeksene i etterkant da 
insert med hele tabellen sammen med indeksering kan ta kjempelang tid.

For å finne skjemadefinisjonene man vil legge kan man se dette ved å kjøre feks:
`pg_dump -h localhost -p 5432 -U dinbruker@nav.no(eller lignende) -d vilkaarsvurdering --schema-only`


## Migrering

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
Eksempel
```
cloud_sql_proxy -enable_iam_login -instances=etterlatte-prod-207c:europe-north1:etterlatte-sakogbehandlinger=tcp:5432 &
psql -h localhost -p 5432 -U migration-user@etterlatte-dev-9b0b.iam sakogbehandlinger -f /data/test.sql
```
OBS: husk å endre instanse basert på miljø.
Dette finner man ved å kjøre `gcloud projects list`.

https://confluence.adeo.no/display/TE/Migreringssteg+for+database

Dette burde samles på ett sted...

#### OBS!
Her kan du få feilmeldinger ala `error: invalid command \N`
Dette er ikke den reelle feilen men kan være at tabellene ikke er riktig laget
eller at man mangler tilgang. feks
` ERROR:  permission denied for table behandling_versjon
psql:/data/test.sql:7662: error: invalid command \.`
Da må man grante `cloudsqliamserviceaccount` mot denne tabellen med rettighetene man trenger.

## Cleanup

Når du er ferdig med migrering kan du kjøre `cleanup_migration.sh` fra lokal maskin

```shell
bash cleanup_migration.sh
```
