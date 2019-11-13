![CLEVER DATA GIT REPO](https://raw.githubusercontent.com/LiCongMingDeShujuku/git-resources/master/0-clever-data-github.png "李聪明的数据库")

# 使用单个作业自动故障转移所有数据库镜像（第1步， 共10步）
#### Automatically Failover All Database Mirrors With A Single Job (01 of 10)
**发布-日期: 2015年10月29日 (评论)**



## Contents

- [中文](#中文)
- [English](#English)
- [SQL Logic](#Logic)
- [Build Info](#Build-Info)
- [Author](#Author)
- [License](#License) 


## 中文
作业：数据库镜像 -自动故障转移所有数据库镜像
注意：为了使这个过程起作用，你必须在主服务器和镜像服务器上具有相同的作业。

## English
JOB:  DATABASE MIRRORS – FAILOVER ALL MIRRORED DATABASES
Note:  For this process to work; you must have the same Job on both Principal & Mirror Servers.

![#](images/screenshot_01.png?raw=true "#")

### STEP 1:    CONFIRM PRIMARY SERVER
The first step, and perhaps the most important allows you to Failover the Mirrored Databases from any Server (Principal or Mirror).   Whenever you run it.  It detects if there are presently any Principal Databases on the server.  If they exist (even just a single database); it will proceed if to the next step.  If not… It will simply get the Partner server name from the Mirror configuration tables and run the same exact Job starting at Step 2.
Note: The first block of code in every step is the SQL Database Mail configuration. This exists in nearly every step so that each steps process can be run outside of the Job if necessary. This also makes absolute sure that an SQL Database Mail profile does indeed exist, and notifications will be sent out. You’ll need to replace the SMTP Server Name, and the MyDomain.com with your companies domain, and finally the @recipients line will need your email address directly (Preferably a distribution group).


### 第1步：确认主服务器
第一步，也许是最重要的步骤。允许你从任何服务器（主体或镜像）故障转移镜像数据库。每当你运行它，它会检测服务器上是否有任何主要数据库。如果它们存在（即使只是一个数据库），它将继续进入下一步。如果不存在，它只是从镜像配置表中获取伙伴服务器名称，并从步骤2开始运行相同的作业。

注意：每个步骤中的第一个代码块是SQL数据库邮件配置。这几乎存在于每个步骤中，以便必要时可以在作业之外运行每个步骤过程。这也绝对确保SQL数据库邮件配置文件确实存在，并且会发送通知。你需要将SMTP服务器名称和MyDomain.com替换为你的公司域，最后@recipients行将需要你的电子邮件地址（最好是通讯组）。

通知逻辑：




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
        -- Enable SQL Database Mail    --启用SQL数据库邮件
        exec master..sp_configure 'show advanced options',1
        reconfigure;
        exec master..sp_configure 'database mail xps',1
        reconfigure;
 
        ----------------------------------------------------------------------
        -- Add a profile        --添加个人资料
 
        execute msdb.dbo.sysmail_add_profile_sp
            @profile_name       = 'SQLDatabaseMailProfile'
        ,   @description        = 'SQLDatabaseMail';
 
        ----------------------------------------------------------------------
        -- Add the account names you want to appear in the email message.
--添加要在电子邮件中显示的帐户名称。
        execute msdb.dbo.sysmail_add_account_sp
            @account_name       = 'sqldatabasemail@MyDomain.com'
        ,   @email_address      = 'sqldatabasemail@MyDomain.com'
        ,   @mailserver_name    = 'MySMTPServer.MyDomain.com'  
        --, @port               = ####              --optional
        --, @enable_ssl         = 1                 --optional
        --, @username           ='MySQLDatabaseMailProfile'     --optional
        --, @password           ='MyPassword'           --optional
 
        -- Adding the account to the profile  --将帐户添加到个人资料

        execute msdb.dbo.sysmail_add_profileaccount_sp
            @profile_name       = 'SQLDatabaseMailProfile'
        ,   @account_name       = 'sqldatabasemail@MyDomain.com'
        ,   @sequence_number    = 1;
 
        -- Give access to new database mail profile (DatabaseMailUserRole)  
--授予访问新数据库邮件配置文件的权限（DatabaseMailUserRole）
        execute msdb.dbo.sysmail_add_principalprofile_sp
            @profile_name       = 'SQLDatabaseMailProfile'
        ,   @principal_id       = 0
        ,   @is_default     = 1;
 
        ----------------------------------------------------------------------
        -- Get Server info for test message  --获取测试消息的服务器信息
 
        declare @get_basic_server_name              varchar(255)
        declare @get_basic_server_name_and_instance_name    varchar(255)
        declare @basic_test_subject_message         varchar(255)
        declare @basic_test_body_message            varchar(max)
        set @get_basic_server_name              = (select cast(serverproperty('servername') as varchar(255)))
        set @get_basic_server_name_and_instance_name    = (select  replace(cast(serverproperty('servername') as varchar(255)), '\', '   SQL Instance: '))
        set @basic_test_subject_message     = 'Test SMTP email from SQL Server: ' + @get_basic_server_name_and_instance_name
        set @basic_test_body_message        = 'This is a test SMTP email from SQL Server:  ' + @get_basic_server_name_and_instance_name + char(10) + char(10) + 'If you see this.  It''s working perfectly :)'
 
        ----------------------------------------------------------------------
        -- Send quick email to confirm email is properly working. --发送快速电子邮件以确认电子邮件是否正常工作
       EXEC msdb.dbo.sp_send_dbmail
            @profile_name       = 'SQLDatabaseMailProfile'
        ,   @recipients     = 'SQLJobAlerts@MyDomain.com'
        ,   @subject        = @basic_test_subject_message
        ,   @body           = @basic_test_body_message;
 
        -- Confirm message send    --确认邮件发送

        -- select * from msdb..sysmail_allitems
    end


```

### 步骤1：逻辑 (Step 1 Logic)

```SQL
use master;
set nocount on
 
----------------------------------------------------------------------------------------
-- confirm principal and mirror servers  --确认主服务器和镜像服务器

declare
    @confirm_mirror_server  varchar(255) = ( select @@servername )
,   @confirm_principal_server   varchar(255) = ( select top 1 replace(left(mirroring_partner_name, charindex('.', mirroring_partner_name) - 1), 'TCP://', '') from master.sys.database_mirroring where mirroring_guid is not null )
,   @instance       varchar(255) = ( select top 1 mirroring_partner_instance from master.sys.database_mirroring where mirroring_guid is not null )
,   @witness        varchar(255) = ( select top 1 case mirroring_witness_name when '' then 'None configured' end from master.sys.database_mirroring where mirroring_guid is not null )
 
----------------------------------------------------------------------------------------
-- create confirmation email message    --创建确认电子邮件
 
declare
    @confirm_server_message_subject varchar(max) = 'You are attempting to run the Mirror Failover Process from the Secondary Server: ' + @confirm_mirror_server + '.  The Failover process is typically run from the Primary Server"' + @confirm_principal_server
,   @confirm_server_message_body    varchar(max) = 'You are attempting to run the Mirror Failover Process from the Secondary Server: ' + @confirm_mirror_server + '.  There are presently no Principal Databases on this Server.  This Job will now cancel on this local server, and execute the Failover process instead from the Primary Server ' + @confirm_principal_server + '. A notification will be sent out automatically from the Primary server when the process begins.'
 
if not exists (select top 1 database_id  from sys.database_mirroring where mirroring_role_desc = 'PRINCIPAL')
    begin
        exec msdb.dbo.sp_send_dbmail
            @profile_name   = 'SQLDatabaseMailProfile'
        ,   @recipients = 'SQLJobAlerts@MyDomain.com'
        ,   @subject    = @confirm_server_message_subject
        ,   @body       = @confirm_server_message_body
 
        waitfor delay '00:00:5';
         
        if not exists(select value from master.sys.configurations where name = 'show advanced options')
            begin
                exec master..sp_configure 'show advanced options', 1; reconfigure with override
            end
 
        if not exists(select value from master.sys.configurations where name = 'xp_cmdshell')
            begin
                exec master..sp_configure 'xp_cmdshell', 1; reconfigure with override
            end
 
        declare
            @retcode        int
        ,   @job_name   varchar(255)
        ,   @step_name  varchar(255)
        ,   @server_name    varchar(255)
        ,   @query      varchar(8000) 
        ,   @cmd        varchar(8000)
        set @job_name   = 'DATABASE MIRRORS - Failover All Mirrored Databases'
        set @step_name      = 'Start Mirror Failover Process'
        set @server_name    = @confirm_principal_server
        set @query      = 'exec msdb.dbo.sp_start_job @job_name = '''   + @job_name + ''', @step_name = ''' + @step_name + ''''
        set @cmd        = 'osql -E -S ' + @server_name + ' -Q "'        + @query + '"'
 
        print ' @job_name = '   +isnull(@job_name,      'NULL @job_name') 
        print ' @server_name = '    +isnull(@server_name,   'NULL @server_name') 
        print ' @query = '      +isnull(@query,     'NULL @query') 
        print ' @cmd = '        +isnull(@cmd,       'NULL @cmd')
 
        exec    @retcode = xp_cmdshell @cmd
 
        if @retcode <> 0 or @retcode is null
            begin
                print 'xp_cmdshell @retcode = '+isnull(convert(varchar(20),@retcode),'NULL @retcode')
            end
 
        raiserror('50005 Mirror Failover Warning.  Mirror Database Failover was initiated from the Mirror Server.  Process will instead be executed on the Primary Server', 16, -1, @@servername )
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

