
# Hadoop/hive start commands

Live Code sharing 

```
https://codepad.pro/pad/cMK5seSubOTc
```




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



## Web interfaces



Open these URLs in the Windows browser while Hadoop is running in WSL:

| Interface | URL | What to inspect |
|---|---|---|
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
