![CLEVER DATA GIT REPO](https://raw.githubusercontent.com/LiCongMingDeShujuku/git-resources/master/0-clever-data-github.png "李聪明的数据库")

# 使用单个作业自动故障转移所有数据库镜像（第6步，共10步
#### Automatically Failover All Database Mirrors With A Single Job (06 of 10)
**发布-日期: 2015年10月29日 (评论)**

![#](images/screenshot_06.png?raw=true "#")

## Contents

- [中文](#中文)
- [English](#English)
- [SQL Logic](#Logic)
- [Build Info](#Build-Info)
- [Author](#Author)
- [License](#License) 


## 中文
作业：数据库镜像 -自动故障转移所有数据库镜像

## English
JOB:  DATABASE MIRRORS – FAILOVER ALL MIRRORED DATABASES

## 步骤6：确认镜像数据库故障
#### STEP 6:  CONFIRM MIRROR DATABASE FAILOVER


逻辑步骤 (Step logic)


---
## Logic
```SQL
use msdb;
set nocount on
set ansi_nulls on
set quoted_identifier on
 
----------------------------------------------------------------------
-- Configure SQL Database Mail if it's not already configured.  
--配置SQL数据库邮件（如果尚未配置）。

if (select top 1 name from msdb..sysmail_profile) is null
    begin
        ----------------------------------------------------------------------
        -- Enable SQL Database Mail  --启用SQL数据库邮件
        exec master..sp_configure 'show advanced options',1
        reconfigure;
        exec master..sp_configure 'database mail xps',1
        reconfigure;
 
        ----------------------------------------------------------------------
        -- Add a profile  --添加个人资料
        execute msdb.dbo.sysmail_add_profile_sp
            @profile_name       = 'SQLDatabaseMailProfile'
        ,   @description        = 'SQLDatabaseMail';
 
        ----------------------------------------------------------------------
        -- Add the account names you want to appear in the email message.
--添加在电子邮件中显示的帐户名称。
        execute msdb.dbo.sysmail_add_account_sp
            @account_name       = 'sqldatabasemail@MyDomain.com'
        ,   @email_address      = 'sqldatabasemail@MyDomain.com'
        ,   @mailserver_name    = 'MySMTPServer.MyDomain.com'  
        --, @port           = ####          --optional
        --, @enable_ssl     = 1         --optional
        --, @username       ='MySQLDatabaseMailProfile' --optional
        --, @password       ='MyPassword'       --optional
 
        -- Adding the account to the profile   --将帐户添加到个人资料
        execute msdb.dbo.sysmail_add_profileaccount_sp
            @profile_name       = 'SQLDatabaseMailProfile'
        ,   @account_name       = 'sqldatabasemail@MyDomain.com'
        ,   @sequence_number    = 1;
 
        -- Give access to new database mail profile (DatabaseMailUserRole)
--给访问新数据库邮件资料授权（DatabaseMailUserRole）
        execute msdb.dbo.sysmail_add_principalprofile_sp
            @profile_name       = 'SQLDatabaseMailProfile'
        ,   @principal_id       = 0
        ,   @is_default     = 1;
 
        ----------------------------------------------------------------------
        -- Get Server info for test message --获取测试消息的服务器信息
 
        declare @get_basic_server_name          varchar(255)
        declare @get_basic_server_name_and_instance_name    varchar(255)
        declare @basic_test_subject_message     varchar(255)
        declare @basic_test_body_message        varchar(max)
        set @get_basic_server_name          = (select cast(serverproperty('servername') as varchar(255)))
        set @get_basic_server_name_and_instance_name    = (select  replace(cast(serverproperty('servername') as varchar(255)), '\', '   SQL Instance: '))
        set @basic_test_subject_message     = 'Test SMTP email from SQL Server: ' + @get_basic_server_name_and_instance_name
        set @basic_test_body_message        = 'This is a test SMTP email from SQL Server:  ' + @get_basic_server_name_and_instance_name + char(10) + char(10) + 'If you see this.  It''s working perfectly :)'
 
        ----------------------------------------------------------------------
-- Send quick email to confirm email is properly working. 
--发送电子邮件以确认电子邮件是否正常工作
       EXEC msdb.dbo.sp_send_dbmail
            @profile_name   = 'SQLDatabaseMailProfile'
        ,   @recipients = 'SQLJobAlerts@MyDomain.com'
        ,   @subject    = @basic_test_subject_message
        ,   @body       = @basic_test_body_message;
 
        -- Confirm message send
--确认邮件发送
        -- select * from msdb..sysmail_allitems
    end
 
use master;
set nocount on
----------------------------------------------------------------------------------------
-- get basic server info.     --获取基本服务器信息
 
declare @server_name_basic      varchar(255)
declare @server_name_instance_name  varchar(255)
set @server_name_basic      = (select cast(serverproperty('servername') as varchar(255)))
set @server_name_instance_name  = (select  replace(cast(serverproperty('servername') as varchar(255)), '\', '   SQL Instance: '))
 
----------------------------------------------------------------------------------------
-- get basic server mirror role info.   --获取基本服务器镜像角色信息。
 
declare
    @primary    varchar(255) = ( select @@servername )
,   @secondary  varchar(255) = ( select top 1 replace(left(mirroring_partner_name, charindex('.', mirroring_partner_name) - 1), 'TCP://', '') from master.sys.database_mirroring where mirroring_guid is not null )
,   @instance   varchar(255) = ( select top 1 mirroring_partner_instance from master.sys.database_mirroring where mirroring_guid is not null )
,   @witness    varchar(255) = ( select top 1 case mirroring_witness_name when '' then 'None configured' end from master.sys.database_mirroring where mirroring_guid is not null )
 
----------------------------------------------------------------------------------------
-- set message subject.     --设置消息主题
declare @message_subject    varchar(255)
set @message_subject    = 'Failover error found on Server:  ' + @server_name_instance_name + '  - Not all databases were properly failed over.'
 
----------------------------------------------------------------------------------------
-- create table to hold confirm failover    
 
if object_id('tempdb..#confirm_mirror_failover') is not null
    drop table #confirm_mirror_failover
 
create table #confirm_mirror_failover
    (
        [server_name]       varchar(255)
    ,   [database]      varchar(255)
    ,   [mirror_state]      varchar(255)
    ,   [mirror_role]       varchar(255)
    ,   [mirror_partner_name]   varchar(255)
    )
 
----------------------------------------------------------------------------------------
-- populate table to hold confirm failover   --填充表以保持确认故障转移

insert into #confirm_mirror_failover
select
    upper(@@servername)
,   upper(DB_NAME(database_id))
,   mirroring_state_desc
,   mirroring_role_desc
,   mirroring_partner_name
from
    sys.database_mirroring
where
    mirroring_guid is not null
    and mirroring_role_desc in ('PRINCIPAL', 'DISCONNECTED', 'SUSPENDED')
 
----------------------------------------------------------------------------------------
-- create conditions for html tables in top and mid sections of email.
 --在电子邮件的顶部和中间部分为html表创建条件。
declare @xml_top            NVARCHAR(MAX)
declare @xml_mid            NVARCHAR(MAX)
declare @body_top           NVARCHAR(MAX)
declare @body_mid           NVARCHAR(MAX)
 
----------------------------------------------------------------------------------------
-- set xml top table td's
-- create html table object for: #check_mirror_latency
 --设置xml顶部表为td's
--创建html表对象为：#check_mirror_latency
set @xml_top = 
    cast(
        (select
            [server_name]       as 'td'
        ,   ''
        ,   [database]      as 'td'
        ,   ''
        ,   [mirror_state]      as 'td'
        ,   ''
        ,   [mirror_role]       as 'td'
        ,   ''
        ,   [mirror_partner_name]   as 'td'
        ,   ''
 
        from  #confirm_mirror_failover
        order by [database], [mirror_role] asc
        for xml path('tr')
        ,   elements)
        as NVARCHAR(MAX)
        )
 
----------------------------------------------------------------------------------------
-- set xml mid table td's
-- create html table object for: #extra_table_formatting_if_needed
--设置xml中部表为td's
--创建html表对象为：# extra_table_formatting_if_needed 
/*
set @xml_mid = 
    cast(
        (select 
            [Column1]as 'td'
        ,   ''
        ,   [Column2]as 'td'
        ,   ''
        ,   [Column3]as 'td'
        ,   ''
        ,   [...]   as 'td'
 
        from  #get_last_known_backups 
        order by [database], [time_of_backup] desc 
        for xml path('tr')
    ,   elements)
    as NVARCHAR(MAX)
        )
*/
----------------------------------------------------------------------------------------
-- format email   --格式化电子邮件
set @body_top =
        '<html>
        <head>
 
<style>
                    h1{
                        font-family: sans-serif;
                        font-size: 87%;
                    }
                    h3{
                        font-family: sans-serif;
                        color: black;
                    }
 
                    table, td, tr, th {
                        font-family: sans-serif;
                        font-size: 87%;
                        border: 1px solid black;
                        border-collapse: collapse;
                    }
                    th {
                        text-align: left;
                        background-color: gray;
                        color: white;
                        padding: 5px;
                        font-size: 87%;
                    }
 
                    td {
                        padding: 5px;
                    }
            </style>
 
        </head>
        <body>
 
<H3>' + @message_subject + '</H3>
 
 
 
<h1>
        A Failover error occurred while failing over from the Primary Server:  <font color="blue">'   + @primary      + ' </font> 
        to the Secondary Server: <font color="blue">' + @secondary        +
                        case
                            when @secondary <> @instance then @secondary + '\' + @instance
                            else ''
                        end + '             </font>
                 
        Not all the Mirrored Datatabases failed over.  Please check the databases on the server, and resolve before proceeding to the next step.
        </h1>
 
          
 
<h1>The following databases did not failover properly.</h1>
 
 
<table border = 1>
 
<tr>
 
<th> Server Name  </th>
 
 
<th> Database     </th>
 
 
<th> Mirror State </th>
 
 
<th> Mirror Role  </th>
 
 
<th> Mirror Partner Name</th>
 
        </tr>
 
'
 
set @body_top = @body_top + @xml_top +
/* '</table>
 
 
<h1>Mid Table Title Here</h1>
 
 
<table border = 1>
 
<tr>
 
<th> Column1 Here </th>
 
 
<th> Column2 Here </th>
 
 
<th> Column3 Here </th>
 
 
<th> ...      </th>
 
        </tr>
 
'       
+ @xml_mid */ 
'</table>
 
 
<h1>Go to the server using Start-Run, or (Win + R) and type in: mstsc -v:' + @server_name_basic + '</h1>
 
'
+ '</body></html>'
 
----------------------------------------------------------------------------------------
-- send email.    --发送电子邮件
 
if exists(select top 1 * from #confirm_mirror_failover)
    begin
        exec msdb.dbo.sp_send_dbmail
            @profile_name       = 'SQLDatabaseMailProfile'
        ,   @recipients     = 'SQLJobAlerts@MyDomain.com'
        ,   @subject        = @message_subject
        ,   @body           = @body_top
        ,   @body_format        = 'HTML';
         
        drop table #confirm_mirror_failover
        raiserror('50005 Mirror Failover Error.  Not all Mirrored Databases were properly failed over.', 16, -1, @@servername )
    end


```



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

