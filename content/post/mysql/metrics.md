
## IO 相关
|指标|查看地方|前提|备注|
|---|---|---|---|
|IO等待|performace_schema.table_io_waits_summary_by_table|setup_instruments.memory/performance_schema/table_io_waits_summary_by_index_usage.enabled=YES||


## 锁相关

|指标|查看地方|前提|备注|
|---|---|---|---|
|锁持有时间|performace_schema.table_handles|setup_instruments.wait/lock/table/sql/handler.enabled=YES|行仅展示了是否持有|
|锁等待|performace_schema.table_lock_waits_summary_by_table|setup_instruments.enabled=YES(count) timed=YES(timed)||

## 主从

|指标|查看地方|前提|备注|
|---|---|---|---|
|处理 relay log 文件数|show slave status.Relay_Log_File or Relay_Master_Log_File|start slave||

## 内存使用


|指标|查看地方|前提|备注|
|---|---|---|---|
|记录 performance_schema 使用的内存|select * from memory_summary_global_by_event_name where event_name like 'memory/performance_schema%'|select enabled from setup_instruments where name like 'memory/performance_schema%' = 'YES'||
|记录 innodb 相关使用的内存|select * from memory_summary_global_by_event_name where event_name like 'memory/innodb%'|select enabled from setup_instruments where name like 'memory/innodb%' = 'YES'|看上去关系很大, 但是在db34上看不到? 待确认|
