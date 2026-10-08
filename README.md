
# Hadoop/hive start commands

Live Code sharing 


https://codepad.pro/pad/cMK5seSubOTc




```
jupyter lab  --notebook-dir=/home/dev/training
```

We have hadoop, spark and hive installed on Linux


```
start-dfs.sh
```
```
start-yarn.sh
```
```
mapred --daemon start historyserver
```

Spark master
```
/opt/spark/sbin/start-master.sh
```

Spark worker

```
/opt/spark/sbin/start-worker.sh "spark://$(hostname):7077"
```


```
nohup hive --service metastore > "$HOME/hive-logs/metastore.log" 2>&1 &
```
```
nohup hiveserver2 > "$HOME/hive-logs/hiveserver2.log" 2>&1 &
```

```
beeline -u 'jdbc:hive2://localhost:10000/default' -n "$USER"
```


### Docker start commands


depends on which directory you have the directory

```
cd training
```


```
docker compose -f postgresql.yaml up -d
```



 

```
docker compose -f polaris.yaml up -d
```

 

```
docker compose -f seaweedfs.yaml up -d
```

 
Check daily

hdfs, spark, hive running or not

```
jps
```

check polaris, seaweedfs running
```
docker ps
```


For S3/Polaris access on CLI

```
export AWS_ACCESS_KEY_ID=team
export AWS_SECRET_ACCESS_KEY=team1234
export AWS_DEFAULT_REGION=us-east-1
```

```
export POLARIS_URL="http://localhost:8181"
```

polaris health check

```
curl -s http://localhost:8182/q/health | jq
```

```
export POLARIS_TOKEN=$(curl -s \
  "$POLARIS_URL/api/catalog/v1/oauth/tokens" \
  --user root:s3cr3t \
  -d 'grant_type=client_credentials' \
  -d 'scope=PRINCIPAL_ROLE:ALL' \
  | jq -r '.access_token')
```

--

Listing catalog test

```
curl -s \
  "$POLARIS_URL/api/management/v1/catalogs" \
  -H "Authorization: Bearer $POLARIS_TOKEN" \
  | jq
```

Token refresh, you may run daily basic even if you get error token or token expired error



## Web interfaces



Open these URLs in the Windows browser while Hadoop is running in WSL:

| Interface | URL | What to inspect |
|---|---|---|
|Seaweed AWS S3| http://localhost:9002 | S3 |
|Airflow| [http://localhost:8090/airflow] (http://localhost:8090/airflow) | Airflow |
|Spark UI | [http://localhost:8080](http://localhost:8080) | Spark UI |
| ResourceManager | [http://localhost:8088](http://localhost:8088) | Applications, states, queues, nodes, memory, and vCores |
| ResourceManager applications | [http://localhost:8088/cluster/apps](http://localhost:8088/cluster/apps) | Running, completed, and failed applications |
| ResourceManager nodes | [http://localhost:8088/cluster/nodes](http://localhost:8088/cluster/nodes) | NodeManager health and available resources |
| NodeManager | [http://localhost:8042](http://localhost:8042) | Containers and local logs on this node |
| MapReduce JobHistory | [http://localhost:19888](http://localhost:19888) | Completed MapReduce jobs, tasks, attempts, counters, and logs |
| NameNode | [http://localhost:9870](http://localhost:9870) | HDFS files, DataNodes, capacity, and cluster storage |

WSL normally forwards listening ports to Windows `localhost`. If a page does not open, confirm that the Hadoop daemons are running:

~~~bash
jps
~~~
