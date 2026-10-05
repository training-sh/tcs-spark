# Quick one

1. create a catalog called movielens [rest api/spark config]
2. create database/namespaces called movie_silver, movie_gold
3. You just need to refactor what you have done in https://github.com/training-sh/tcs-spark/blob/main/04-Polaris/02_polaris_ecommerce_lab.ipynb
4. no new code..



```

open terminal

export POLARIS_URL="http://localhost:8181"

export POLARIS_TOKEN=$(curl -s \
  "$POLARIS_URL/api/catalog/v1/oauth/tokens" \
  --user root:s3cr3t \
  -d 'grant_type=client_credentials' \
  -d 'scope=PRINCIPAL_ROLE:ALL' \
  | jq -r '.access_token')


create a bucket with name movielens in seaweedfs


curl -s \
  -X POST "$POLARIS_URL/api/management/v1/catalogs" \
  -H "Authorization: Bearer $POLARIS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "catalog": {
      "type": "INTERNAL",
      "name": "movielens",
      "properties": {
        "default-base-location": "s3://movielens/"
      },
      "storageConfigInfo": {
        "storageType": "S3",
        "allowedLocations": [
          "s3://movielens/"
        ],
        "endpoint": "http://localhost:9001",
        "endpointInternal": "http://seaweedfs:8333",
        "region": "us-east-1",
        "pathStyleAccess": true,
        "stsUnavailable": true
      }
    }
  }' | jq



```
