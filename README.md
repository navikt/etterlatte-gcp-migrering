# etterlatte-gcp-migrering

Verktøy for å gjennomføre databasemigrering mellom Google Cloud SQL databaser. 

## Forarbeid

### 1. Start app for migrering

Start pod'en som inneholder verktøyene ved å applye til ditt namespace:
```
kubectl apply -f gcloud.yaml
```

Obs merk at denne er begrenset til maks 512 mb.
Om man deployer med en ugydlig config må man sjekke loggene til applikasjonsressursen evt serviceressursen som blir opprettet i 
tillegg til podden.

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

Ip'ene som refereres må peke på riktige public ip'er for databasene som skal aksesseres. Dette finner man i oversikten over 
databaseinstanser i GCP Console. 

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

For at det skal være mulig for serviceaccount-brukeren å lese fra databasen som det skal gjøres dump av, må det gis tilganger
til skjemaet og til tabellene i denne databasen:

```postgresql
GRANT USAGE on SCHEMA public to "<SERVICEACCOUNT-USER>";
GRANT SELECT ON ALL TABLES IN SCHEMA public TO "<SERVICEACCOUNT-USER>";
```

OBS: Serviceaccount brukeren må også legges inn manuelt i brukerseksjonen på cloud console ellers vil man få auth feil når man connecter mot databasen
"Add user account" her https://console.cloud.google.com/sql/instances/etterlatte-sakogbehandlinger/users?authuser=1&inv=1&invt=AbqdIQ&project=etterlatte-prod-207c&rapt=AEjHL4O5-Yd1XwWAQJvYDtURcVk44X4RtujH_D8K3TA8gMSaFNxK6udNwQf9GmGEsBAV2J6kQcOl44r-_nyKyOCAXiLSe8OgTY17CDf946Z_oiRqHj4cGhs
Her må man legge inn hele quailified name på service account brukeren altså `migrering-user@etterlatte-prod-207c.iam.gserviceaccount.com`
`migration-user` vs `migrering-user`
Ellers får man: `pg_dump: error: connection to server at "localhost" (127.0.0.1), port 5432 failed: FATAL:  password authentication failed for user "migration-user@etterlatte-prod-207c.iam"`

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
