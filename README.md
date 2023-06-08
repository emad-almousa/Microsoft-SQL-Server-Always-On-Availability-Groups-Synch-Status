Microsoft SQL Server Always-On Availability Groups Synch Status

You can configure a database job to check synchronization status every 1 hour to check if syncrhonization is healthy or not.  This is important to ensure the following when synch-mode between replica nodes are in-place:

Log Backups are taken successfully from primary node and avoid unncessary outages
Ensure swift failover without data loss



The main script building block is:


DECLARE @synchronization_state_desc_flag varchar(max)

 

              SET @synchronization_state_desc_flag = (SELECT distinct(drs.synchronization_state_desc)

              FROM sys.dm_hadr_database_replica_states AS drs

INNER JOIN sys.availability_databases_cluster AS adc

              ON drs.group_id = adc.group_id AND

              drs.group_database_id = adc.group_database_id

INNER JOIN sys.availability_groups AS ag

              ON ag.group_id = drs.group_id

INNER JOIN sys.availability_replicas AS ar

              ON drs.group_id = ar.group_id AND

              drs.replica_id = ar.replica_id where drs.synchronization_state_desc='NOT SYNCHRONIZING')

 

IF ( @synchronization_state_desc_flag='NOT SYNCHRONIZING')

   EXEC msdb.dbo.sp_send_dbmail

    @profile_name = 'DBA',

    @recipients = 'DBATEAM@CONTOSO.com',

    @body = 'There is a porblem in synchronization between Always-On Nodes and requires your action in Nodes XXXX,XXXX',

    @subject = '**ALERT**- Always-On Availability Group Snynchronization status is not healthy'
    
    
    **********************************************************************************************************************************
                                            SQL Server Job Code
                                           
    ************************************************************************************************************************************
    
    USE [msdb]
GO

BEGIN TRANSACTION
DECLARE @ReturnCode INT
SELECT @ReturnCode = 0
IF NOT EXISTS (SELECT name FROM msdb.dbo.syscategories WHERE name=N'[Uncategorized (Local)]' AND category_class=1)
BEGIN
EXEC @ReturnCode = msdb.dbo.sp_add_category @class=N'JOB', @type=N'LOCAL', @name=N'[Uncategorized (Local)]'
IF (@@ERROR <> 0 OR @ReturnCode <> 0) GOTO QuitWithRollback

END

DECLARE @jobId BINARY(16)
EXEC @ReturnCode =  msdb.dbo.sp_add_job @job_name=N'Always-On-Replicas-Health-Detection', 
		@enabled=1, 
		@notify_level_eventlog=0, 
		@notify_level_email=0, 
		@notify_level_netsend=0, 
		@notify_level_page=0, 
		@delete_level=0, 
		@description=N'No description available.', 
		@category_name=N'[Uncategorized (Local)]', 
		@owner_login_name=N'sa', @job_id = @jobId OUTPUT
IF (@@ERROR <> 0 OR @ReturnCode <> 0) GOTO QuitWithRollback
/****** Object:  Step [health_script]    Script Date: 08/06/2023 21:19:37 ******/
EXEC @ReturnCode = msdb.dbo.sp_add_jobstep @job_id=@jobId, @step_name=N'health_script', 
		@step_id=1, 
		@cmdexec_success_code=0, 
		@on_success_action=1, 
		@on_success_step_id=0, 
		@on_fail_action=2, 
		@on_fail_step_id=0, 
		@retry_attempts=0, 
		@retry_interval=0, 
		@os_run_priority=0, @subsystem=N'TSQL', 
		@command=N'DECLARE @synchronization_state_desc_flag varchar(max)

 

              SET @synchronization_state_desc_flag = (SELECT distinct(drs.synchronization_state_desc)

              FROM sys.dm_hadr_database_replica_states AS drs

INNER JOIN sys.availability_databases_cluster AS adc

              ON drs.group_id = adc.group_id AND

              drs.group_database_id = adc.group_database_id

INNER JOIN sys.availability_groups AS ag

              ON ag.group_id = drs.group_id

INNER JOIN sys.availability_replicas AS ar

              ON drs.group_id = ar.group_id AND

              drs.replica_id = ar.replica_id where drs.synchronization_state_desc=''NOT SYNCHRONIZING'')

 

IF ( @synchronization_state_desc_flag=''NOT SYNCHRONIZING'')

   EXEC msdb.dbo.sp_send_dbmail

    @profile_name = ''DBA'',

    @recipients = ''DBATEAM@CONTOSO.com'',

    @body = ''There is a porblem in synchronization between Always-On Nodes and requires your action in Nodes XXXX,XXXX'',

    @subject = ''**ALERT**- Always-On Availability Group Snynchronization status is not healthy''




', 
		@database_name=N'master', 
		@flags=0
IF (@@ERROR <> 0 OR @ReturnCode <> 0) GOTO QuitWithRollback
EXEC @ReturnCode = msdb.dbo.sp_update_job @job_id = @jobId, @start_step_id = 1
IF (@@ERROR <> 0 OR @ReturnCode <> 0) GOTO QuitWithRollback
EXEC @ReturnCode = msdb.dbo.sp_add_jobschedule @job_id=@jobId, @name=N'hourly_check', 
		@enabled=1, 
		@freq_type=4, 
		@freq_interval=1, 
		@freq_subday_type=8, 
		@freq_subday_interval=1, 
		@freq_relative_interval=0, 
		@freq_recurrence_factor=0, 
		@active_start_date=20230608, 
		@active_end_date=99991231, 
		@active_start_time=0, 
		@active_end_time=235959, 
		@schedule_uid=N'eca39fbf-0f1f-448a-8352-47d976352bc3'
IF (@@ERROR <> 0 OR @ReturnCode <> 0) GOTO QuitWithRollback
EXEC @ReturnCode = msdb.dbo.sp_add_jobserver @job_id = @jobId, @server_name = N'(local)'
IF (@@ERROR <> 0 OR @ReturnCode <> 0) GOTO QuitWithRollback
COMMIT TRANSACTION
GOTO EndSave
QuitWithRollback:
    IF (@@TRANCOUNT > 0) ROLLBACK TRANSACTION
EndSave:
GO





    
    
    
    
    
    
    
    
    
    
    
    
    
    
    
    
    
    
    
    
    
    
    
    
    
    
    
    
