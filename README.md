# gcp-migrering

Verktøy for å gjennomføre databasemigrering fra on-premises databaser til Google Cloud SQL.

Les mer om migrering med `pg_dump` hos [docs.nais.io](https://docs.nais.io/clusters/migrating-databases-to-gcp/#from-on-premise-postgresql).

## Bruk
Start pod'en som inneholder verktøyene ved å applye til ditt namespace: 
```
kubectl apply -f gcloud.yaml
```

Exec inn i pod'en for å kjøre ønskede kommandoer:
```
kubectl exec -it <navn på pod> -- sh
```

## Innhold
[gcloud pod spec](./gcloud.yaml) er pod spec'en som innholder alle verktøy man trenger. Denne er bygget fra [gcloud-psql repository](https://github.com/nais/gcloud-psql). Pod'en har ikke lenger et fysisk volum mountet da dette ikke er støttet on-premises, så dersom man skal eksportere store databaser må man benytte en annen metode. Disk'en tilgjengelig er oppad begrenset til det nodenden kjører på  har tilgjengelig.

[skript for opprydding](./cleanup-migration.sh) sletter pod og secret.

[secret](./secret.yaml) kan brukes for å lagre hemmeligheten man benytter for å koble til gcloud/gcp i clusteret. I [gcloud pod spec](./gcloud.yaml) er det referert til navnet på hemmeligheten (migration-user), så denne kubernetes hemmeligheten må hete det samme, evt må man også endre i pod spec.
