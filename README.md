![CLEVER DATA GIT REPO](https://raw.githubusercontent.com/LiCongMingDeShujuku/git-resources/master/0-clever-data-github.png "李聪明的数据库")

# 自动为无人参与配置文件生成SQL实例名称
#### Automatically Generate SQL Instance Name For Unattended Config File
**发布-日期: 2016年09月09日 (评论)**

### 以下是使用此方法创建的实例。
#### Here are the instances that were created with this approach.

![#](images/automatically-add-shared-instances.png?raw=true "#")

## Contents

- [中文](#中文)
- [English](#English)
- [SQL Logic](#Logic)
- [Build Info](#Build-Info)
- [Author](#Author)
- [License](#License) 


## 中文
如果你要创建无人参与的安装过程，并希望将自己的参数连接到配置文件中，那么你可以通过以下方式做到。
我为这个过程做了很多假设，并且提供了一些通用参数，但是我已经对其他配置运行了几个测试，这个逻辑处于完美的工作状态。我已经自动创建了几个实例。

这显然是为多sql服务器环境设计的，其中一个服务器下有许多实例。
在此示例中，实例名称具有以下框架，并且下面的逻辑确保创建的其他每个实例遵循此约定。
SQLSHARE01，... 02，... 03等

## English
If you’re looking to create an unattended installation process, and want to concatenate your own parameters into the configuration file; then here is one way you can do it.
There are a number of assumptions that are made for this process, and there are some generic parameters supplied, but I’ve run several tests around other configurations and this logic is in perfect working order. I’ve created several instances automatically.
This is obviously designed for multi-sql server environments where there are many instances under one server.
In this example the names instances have the following framework, and the logic below ensures this convention will be maintained for each additional instance that is created.
SQLSHARE01, …02, …03 etc.


逻辑如下(Here’s the logic)


---
## Logic
```SQL
use master;
set nocount on
 
declare @server_name    varchar(255) = (select @@servername)
declare @sql_service    varchar(255) = 'MyDomain\MySQLServiceAccount'
declare @sql_service_pw varchar(255) = 'MyPassword'
declare @sql_agent  varchar(255) = 'MyDomain\MyAgentServiceAccount'
declare @sql_agent_pw   varchar(255) = 'MyPassword'
declare @dba_group  varchar(255) = 'MyDomain\MyDBAGroup'
declare @sql_sa     varchar(255) = 'MysaAccount'
declare @sql_sa_pw  varchar(255) = 'MysaPassword'
declare @my_user_data   varchar(255) = 'MyUserDataPath'
declare @my_user_log    varchar(255) = 'MyUserLogPath'
declare @my_temp_data   varchar(255) = 'MyTempDataPath'
declare @my_temp_log    varchar(255) = 'MyTempLogPath'
declare @my_backup_path varchar(255) = 'MyBackupPath'
declare @run_bcp    varchar(8000) 
declare @silent_string  varchar(max)
 
-- create table to hold instance names  --创建表来保存实例名称

declare @sql_instances table
(
    [id]        int identity(0,1)
,   [rootkey]       nvarchar(255)
,   [sql_instances] nvarchar(255)
,   [value]     nvarchar(255)
)
  
-- get instances names using xp_regread     --使用xp_regread获取实例名称

insert into @sql_instances ([rootkey], [sql_instances], [value])
execute xp_regread
    @rootkey    = 'hkey_local_machine'
,   @key        = 'software\microsoft\microsoft sql server'
,   @value_name = 'installedinstances'
  
-- set prefix name for multi-instance environment aka: Shared SQL Environment "SQLSHARE"
-- produce the next instance name.
--设置多实例环境的前缀名称：Shared SQL Environment "SQLSHARE"
--生成下一个实例名称。
declare @prefix     varchar(255) = 'SQLSHARE'
declare @next_instance  varchar(255) = 
(
select
    case
    when max([sql_instances]) is null then 'SQLSHARE01'
    when max([sql_instances]) = 'MSSQLSERVER' then 'SQLSHARE01'
    else @prefix + cast(cast(right(max([sql_instances]), 2) as int) + 1 as varchar)
    end as [sql_instances]
from
    @sql_instances si
where
    si.[id] > 0
)
-- reference: https://msdn.microsoft.com/en-us/library/ms144259(v=sql.120).aspx
 
set @silent_string          =
'[OPTIONS]
ACTION              ="Install"
ENU             ="True"
QUIET               ="False"
QUIETSIMPLE         ="True"
UpdateEnabled           ="False"
ERRORREPORTING          ="False"
USEMICROSOFTUPDATE      ="False"
FEATURES            =SQLENGINE,RS
UpdateSource            ="MU"
HELP                ="False"
INDICATEPROGRESS        ="False"
X86             ="False"
INSTALLSHAREDDIR        ="E:\Program Files\Microsoft SQL Server"
INSTALLSHAREDWOWDIR     ="E:\Program Files (x86)\Microsoft SQL Server"
INSTANCENAME            ="' + @next_instance + '"
SQMREPORTING            ="False"
INSTANCEID          ="' + @next_instance + '"
RSINSTALLMODE           ="FilesOnlyMode"
INSTANCEDIR         ="E:\Program Files\Microsoft SQL Server"
AGTSVCACCOUNT           ="MyDomain\MyServiceAccount"
AGTSVCPASSWORD          ="MyPassword"
AGTSVCSTARTUPTYPE       ="Automatic"
COMMFABRICPORT          ="0"
COMMFABRICNETWORKLEVEL          ="0"
COMMFABRICENCRYPTION            ="0"
MATRIXCMBRICKCOMMPORT           ="0"
SQLSVCSTARTUPTYPE       ="Automatic"
FILESTREAMLEVEL         ="0"
ENABLERANU          ="False"
SQLCOLLATION            ="SQL_Latin1_General_CP1_CI_AS"
SQLSVCACCOUNT           ="MyDomain\MyServiceAccount"
SQLSVCPASSWORD          ="MyPassword"
AGTSVCACCOUNT           ="MyDomain\MyServiceAccount"
AGTSVCPASSWORD          ="MyPassword"
SQLSYSADMINACCOUNTS     ="MyDomain\MyServiceAccount" "MyDomain\MyDBAGroup"
SECURITYMODE            ="SQL"
SAPWD               ="##MysaPassword' + right(@next_instance, 2) + '!!##"
SQLTEMPDBDIR            ="E:\Program Files\Microsoft SQL Server\MSSQL12.' + @next_instance + '\MSSQL\Data"
SQLTEMPDBLOGDIR         ="E:\Program Files\Microsoft SQL Server\MSSQL12.' + @next_instance + '\MSSQL\Data"
SQLUSERDBDIR            ="E:\Program Files\Microsoft SQL Server\MSSQL12.' + @next_instance + '\MSSQL\Data"
SQLUSERDBLOGDIR         ="E:\Program Files\Microsoft SQL Server\MSSQL12.' + @next_instance + '\MSSQL\Data"
SQLBACKUPDIR            ="E:\Program Files\Microsoft SQL Server\MSSQL12.' + @next_instance + '\MSSQL\Backup"
ADDCURRENTUSERASSQLADMIN        ="False"
TCPENABLED          ="1"
NPENABLED           ="0"
BROWSERSVCSTARTUPTYPE           ="Automatic"
RSSVCACCOUNT            ="MyDomain\MyServiceAccount"
RSSVCPASSWORD           ="MyPassword"
RSSVCSTARTUPTYPE        ="Automatic"'
 
if object_id('tempdb..##silent_string') is not null drop table ##silent_string
create table ##silent_string ([bcpcommand]  varchar(8000)) insert into  ##silent_string select @silent_string
 
select  @run_bcp = 
'bcp "select [bcpcommand] from ##silent_string" ' + 'queryout "e:\sql_silent_install_config_file\sql_2014_config_file.ini" -c -t -T -S' + @server_name 
exec master..xp_cmdshell @run_bcp


```

在你创建配置文件之后，你自然会去检查一下。每当导出某些东西时，标签和间距并不总是那么清晰，所以当你看到它时，它看起来会很乱。选项卡和空格都很乱，但不用担心。操作系统可以自行选择，然后运行安装。只要该部分有效，你就可以了。每次运行，它会重新创建一个.ini文件，所以不管事后发生什么事情，它都将被复制。

当然只要你准备好运行它，只需打开命令提示符（或将其合并到作业步骤中），然后运行以下脚本。你需要确保已经提供了安装介质所在的适当路径。

E:DEPENDENCIESSQL_SERVER_2014STANDARD_X64setup.exe /configurationfile=E:SQL_SILENT_INSTALL_CONFIG_FILEsql_2014_config_file.ini /IacceptSQLServerLicenseTerms

不要忘记标准版支持大约16个实例，而Enterprise可以支持最多50个。添加一些逻辑来查看版本支持的实例数是16或50应该不会很难。


---

Ok; so after you create the configuration file; you’ll naturally be compelled to check it out. Tabs, and spacing does not always look so sharp whenever something is exported so when you see it; it’s going to look butchered. Tabs, and spaces all a mess, but no worries it’s only there so the OS can pick it up, and run through the install. As long as that part works you’ll be fine. Each time this is run; it will recreate the former .ini file so no matter what happens to the file after the fact; it will be reproduced.

Of course; whenever you’re ready to run this, simply open Command Prompt (or incorporate this into a Job step), and run the following script. Make sure to supply the appropriate path where your installation medium exists.

E:DEPENDENCIESSQL_SERVER_2014STANDARD_X64setup.exe /configurationfile=E:SQL_SILENT_INSTALL_CONFIG_FILEsql_2014_config_file.ini /IacceptSQLServerLicenseTerms

Don’t forget people… Standard Edition supports about 16 Instances, while Enterprise can support up to 50. Wouldn’t be too hard to add some logic to check on the versioning and simply stop at 16, or 50 depending on the edition, licensing etc.

---

[![WorksEveryTime](https://forthebadge.com/images/badges/60-percent-of-the-time-works-every-time.svg)](https://shitday.de/)

## Build-Info

| Build Quality | Build History |
|--|--|
|<table><tr><td>[![Build-Status](https://ci.appveyor.com/api/projects/status/pjxh5g91jpbh7t84?svg?style=flat-square)](#)</td></tr><tr><td>[![Coverage](https://coveralls.io/repos/github/tygerbytes/ResourceFitness/badge.svg?style=flat-square)](#)</td></tr><tr><td>[![Nuget](https://img.shields.io/nuget/v/TW.Resfit.Core.svg?style=flat-square)](#)</td></tr></table>|<table><tr><td>[![Build history](https://buildstats.info/appveyor/chart/tygerbytes/resourcefitness)](#)</td></tr></table>|

## Author

- **李聪明的数据库 Lee's Clever Data**
- **Mike的数据库宝典 Mikes Database Collection**
- **李聪明的数据库** "Lee Songming"

[![Gist](https://img.shields.io/badge/Gist-李聪明的数据库-<COLOR>.svg)](https://gist.github.com/congmingshuju)
[![Twitter](https://img.shields.io/badge/Twitter-mike的数据库宝典-<COLOR>.svg)](https://twitter.com/mikesdatawork?lang=en)
[![Wordpress](https://img.shields.io/badge/Wordpress-mike的数据库宝典-<COLOR>.svg)](https://mikesdatawork.wordpress.com/)

---
## License
[![LicenseCCSA](https://img.shields.io/badge/License-CreativeCommonsSA-<COLOR>.svg)](https://creativecommons.org/share-your-work/licensing-types-examples/)

![Lee Songming](https://raw.githubusercontent.com/LiCongMingDeShujuku/git-resources/master/1-clever-data-github.png "李聪明的数据库")

