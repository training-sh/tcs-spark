```
83.149.9.216 - - [17/May/2015:10:05:03 +0000] "GET /presentations/logstash-monitorama-2013/images/kibana-search.png HTTP/1.1" 200 203023 "http://semicomplete.com/presentations/logstash-monitorama-2013/" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_9_1) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/32.0.1700.77 Safari/537.36"

ip_address string => geo dns
event_time datetime
url string
http_method string
status  int
content_length 
referrer 
user_agent 
 

Database/datawarehouse, $$$$$ for talent, managing tools etc
       Highly structured
       Table, schema, data that fit schema data (rows)

	to store this log files, what you do?

	   convert into structured / ETL - Extract Transform Load

	Extract - read log line, parse the data
	Transform  - convert itno a format like json, map ip address to geo dns/location/state/city/..
	Load - Load into MySQL/PostgreSQL


If you want to store the same log file into datalake, what is needed? $ - store the files
	1 TB - 20$ per month
        dump the data - simple
	no need to do ETL ahead


	Extract/Load/Transform

	Load/Extract/Transform



Data Zones
	raw/arrival zone/landing zone
	Medallion architecture bronze -> silver -> gold -> titanium etc






```
