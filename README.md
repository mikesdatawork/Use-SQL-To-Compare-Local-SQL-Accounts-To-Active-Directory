![MIKES DATA WORK GIT REPO](https://raw.githubusercontent.com/mikesdatawork/images/master/git_mikes_data_work_banner_01.png "Mikes Data Work")        

# Use SQL To Compare Local SQL Accounts To Active Directory
**Post Date: March 21, 2018**        



## Contents    
- [About Process](##About-Process)  
- [SQL Logic](#SQL-Logic)  
- [Build Info](#Build-Info)  
- [Author](#Author)  
- [License](#License)       

## About-Process


![compare sql security to active directory]( https://mikesdatawork.files.wordpress.com/2018/03/image0012.png "SQL to AD Linked Server")
 

<p>In most environment DBA' will need to know what accounts have been disabled on the network so they can in-turn be removed from SQL Server as a matter of standard security.

This script will help build the following objects…
Create Linked Server ADSI_LINK
Query Active Directory
Create Temp Table With Active Directory Results
Get Size of Temp Table
Compare Local Domain Accounts to Active Directory
Find Disabled (or deprovisioned accounts)

This will create alist of known Disabled accounts that were added to SQL Server.</p>      


## SQL-Logic
```SQL
use master;
set nocount on
 
-- Create ADSI Linked Server for Active Directory queries (if does not exist)
if (select [srvname] from master..sysservers where [srvname] = 'ADSI_LINK') is null
    begin
        exec master.dbo.sp_addlinkedserver 
            @server         = N'ADSI_LINK'
        ,   @srvproduct     = N'Active Directory Services Interfaces'
        ,   @provider       = N'ADSDSOObject'
        ,   @datasrc        = N'MyDomainController.domain.com';
     
        exec master.dbo.sp_addlinkedsrvlogin 
            @rmtsrvname     = N'ADSI_LINK'
        ,   @useself        = N'False'
        ,   @locallogin     = NULL
        ,   @rmtuser        = N'MyDomain\MyAdminAccount'
        ,   @rmtpassword    = 'MyPassword';
    end
 
-- build variables for Active Directory query
declare @characters varchar(255); 
declare @query      varchar(max); 
declare @attributes varchar(max);
set @characters = 'ABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789';
set @attributes = 
    '   cn
    ,   sAMAccountName
    ,   accountExpires
    ,   pwdLastSet
    ,   userAccountControl
    ,   ADsPath
    ,   lockouttime
    ,   mail
    ,   createTimeStamp
    ,   employeeID
    ,   lastLogon
    ,   co
    ,   l';
set @query = 'SELECT * INTO ##ADRESULTS FROM (';
  
-- build Active Direcotry query
with numbertable as (select top (len(@characters)) rownumb = row_number() over (order by [object_id]) from sys.all_objects order by rownumb)
select @query = @query + 
    'SELECT top 901 * FROM OPENQUERY(ADSI_LINK, ''SELECT ' + @attributes + 
    ' FROM ''''LDAP://DC=MyDomain,DC=com'''' WHERE objectCategory=''''Person'''' AND (cn = ''''' + 
    substring(@characters, numbertable.rownumb, 1) + '*'''') AND (objectClass = ''''user'''' OR objectClass = ''''contact'''')'')
    UNION
    '
from numbertable;
  
-- create final query
select @query = left(@query, len(@query) - charindex(reverse('union'), reverse(@query)) - 4) + ') as query'
  
-- remove if already exists (before first run)
if object_id('tempdb.dbo.##adresults', 'u') is null
    begin
        execute(@query)
    end
 
-- get size of temp table ##adresults_table_created'    = st.name
,   'row_count'     = sddps.row_count
,   'used_size_kb'  = sddps.used_page_count * 8
,   'reserved_kb'   = sddps.reserved_page_count * 8
from 
    tempdb.sys.partitions sp inner join tempdb.sys.dm_db_partition_stats sddps
    on sp.partition_id      = sddps.partition_id 
    and sp.partition_number = sddps.partition_number 
    inner join tempdb.sys.tables as st 
    on sddps.object_id      = st.object_id 
where
    st.[name] = '##adresults'
 
-- get install date to avoid default logins
declare @install_date       datetime = (select [createdate] from syslogins where [sid] = 0x010100000000000512000000)
declare @default_accounts   datetime = (select dateadd(minute, 1, @install_date))
 
-- create table for local logins
if object_id('tempdb..##logins') is not null
    drop table  ##logins
create table    ##logins ([local_logins] varchar(255))
insert into     ##logins select [name] from syslogins where [createdate] > @default_accounts and [name] like '%\%'
 
-- find deprovisioned accounts
select
    'Deprovisioned_Accounts' = 'MyDomain\' + [samaccountname]
from
    [##adresults]
where
    'MyDomain\' + [samaccountname] in (select [local_logins] from ##logins)
    and [userAccountControl] = '514'
 
 
/*
 drop table ##adresults;
 drop table ##logins;
*/

```


[![WorksEveryTime](https://forthebadge.com/images/badges/60-percent-of-the-time-works-every-time.svg)](https://shitday.de/)

## Build-Info

| Build Quality | Build History |
|--|--|
|<table><tr><td>[![Build-Status](https://ci.appveyor.com/api/projects/status/pjxh5g91jpbh7t84?svg?style=flat-square)](#)</td></tr><tr><td>[![Coverage](https://coveralls.io/repos/github/tygerbytes/ResourceFitness/badge.svg?style=flat-square)](#)</td></tr><tr><td>[![Nuget](https://img.shields.io/nuget/v/TW.Resfit.Core.svg?style=flat-square)](#)</td></tr></table>|<table><tr><td>[![Build history](https://buildstats.info/appveyor/chart/tygerbytes/resourcefitness)](#)</td></tr></table>|

## Author

[![Gist](https://img.shields.io/badge/Gist-MikesDataWork-<COLOR>.svg)](https://gist.github.com/mikesdatawork)
[![Twitter](https://img.shields.io/badge/Twitter-MikesDataWork-<COLOR>.svg)](https://twitter.com/mikesdatawork)
[![Wordpress](https://img.shields.io/badge/Wordpress-MikesDataWork-<COLOR>.svg)](https://mikesdatawork.wordpress.com/)

     
## License
[![LicenseCCSA](https://img.shields.io/badge/License-CreativeCommonsSA-<COLOR>.svg)](https://creativecommons.org/share-your-work/licensing-types-examples/)

![Mikes Data Work](https://raw.githubusercontent.com/mikesdatawork/images/master/git_mikes_data_work_banner_02.png "Mikes Data Work")

