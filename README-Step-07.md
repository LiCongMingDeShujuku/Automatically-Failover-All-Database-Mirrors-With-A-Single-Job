![CLEVER DATA GIT REPO](https://raw.githubusercontent.com/LiCongMingDeShujuku/git-resources/master/0-clever-data-github.png "李聪明的数据库")

# 使用单个作业自动故障转移所有数据库镜像（第7步，共10步）
#### Automatically Failover All Database Mirrors With A Single Job (07 of 10)
**发布-日期: 2015年10月29日 (评论)**

![#](images/screenshot_07a.png?raw=true "#")

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


## 步骤7：在PARTNER SERVER上设置高性能模式
**注意操作步骤（在成功和失败时）应遵循如下所示步骤。**
步骤7之后的步骤将独立运行，因此在步骤7完成后它们不能（也不应该）运行。这些步骤由伙伴服务器上的相同作业启动，反之亦然。


#### STEP 7:  SET HIGH PERFORMANCE MODE ON PARTNER SERVER
**It’s important to note that the Step actions ( On Success & On Failure )**should go as follows for this this step.
The steps following 7 are to be run independantly so they cannot (and should not) run after Step 7 has completed. These steps are initiated by the same Job on the Partner Server and vice versa.



![#](images/screenshot_07b.png?raw=true "#")



逻辑步骤 (Step logic)

---
## Logic
```SQL
use master;
set nocount on
 
if not exists(select value from master.sys.configurations where name = 'show advanced options')
    begin
        exec master..sp_configure 'show advanced options', 1; reconfigure with override
    end
 
if not exists(select value from master.sys.configurations where name = 'xp_cmdshell')
    begin
        exec master..sp_configure 'xp_cmdshell', 1; reconfigure with override
    end
 
----------------------------------------------------------------------------------------
-- confirm all servers failed over (only mirrors should exist on this local server at this stage) thus the partner server is the Principal where the HIGH PERFORMANCE mode should be set.
--确认所有服务器都已故障转移（此阶段只有镜像应存在此本地服务器上），因此伙伴服务器是应设置HIGH PERFORMANCE模式的Principal。
-- this was confirmed by the former step (Confirm Mirror Failover).  If this step was reached; it passed otherwise it would fail before this step.  The 'if exists' is added so this logic block so it
--这是前一步（确认镜像故障转移）确认的。如果达到这个步骤将会通过，否则它会在此步骤之前失败。添加了'if exists'，这个逻辑块就这样了。

-- could exist outside of this job step flow if necessary.
--如有必要，可以在此工作步骤流程之外存在。

 
if not exists (select top 1 database_id  from sys.database_mirroring where mirroring_role_desc = 'PRINCIPAL')
    begin
        declare
            @new_principal  varchar(255) 
        ,   @retcode        int
        ,   @job_name   varchar(255)
        ,   @step_name  varchar(255)
        ,   @server_name    varchar(255) 
        ,   @query      varchar(8000) 
        ,   @cmd        varchar(8000)
        set @new_principal  = ( select top 1 replace(left(mirroring_partner_name, charindex('.', mirroring_partner_name) - 1), 'TCP://', '') from master.sys.database_mirroring where mirroring_guid is not null )
        set @job_name   = 'DATABASE MIRRORS - Failover All Mirrored Databases'
        set     @step_name  = 'Set High Performance'
        set     @server_name    = @new_principal
        set     @query      = 'exec msdb.dbo.sp_start_job @job_name = '''   + @job_name + ''', @step_name = ''' + @step_name + ''''
        set     @cmd        = 'osql -E -S ' + @server_name + ' -Q "'    + @query + '"'
 
        print ' @job_name = '   +isnull(@job_name,      'NULL @job_name') 
        print ' @server_name = '    +isnull(@server_name,   'NULL @server_name') 
        print ' @query = '      +isnull(@query,     'NULL @query') 
        print ' @cmd = '        +isnull(@cmd,       'NULL @cmd')
 
        --exec  @retcode = xp_cmdshell @cmd
 
        if @retcode <> 0 or @retcode is null
            begin
                print 'xp_cmdshell @retcode = '+isnull(convert(varchar(20),@retcode),'NULL @retcode')
            end
    end


```

[![WorksEveryTime](https://forthebadge.com/images/badges/60-percent-of-the-time-works-every-time.svg)](https://shitday.de/)

--- 

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

