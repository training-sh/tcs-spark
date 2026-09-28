# Polaris start

```
docker network create dataeng-network


docker compose -f postgresql.yaml up -d


docker compose -f postgresql.yaml ps

docker compose -f polaris.yaml up -d

docker compose -f polaris.yaml ps


docker compose -f seaweedfs.yaml up -d

docker compose -f seaweedfs.yaml ps

docker compose -f seaweedfs.yaml logs -f

```
----

```
docker compose -f seaweedfs.yaml down
docker compose -f polaris.yaml down
docker compose -f postgresql.yaml down
```
----

docker compose -f polaris.yaml up -d

docker compose -f polaris.yaml ps

docker compose -f polaris.yaml logs -f polaris

docker compose -f polaris.yaml logs polaris-bootstrap

docker compose -f polaris.yaml down

docker compose -f polaris.yaml down -v

docker network inspect dataeng-network | jq '.[0].Containers'
docker compose -f polaris.yaml up  --force-recreate
