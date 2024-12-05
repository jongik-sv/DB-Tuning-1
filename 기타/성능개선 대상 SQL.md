```sql
-- IO/CPU TOP SQL 관련 성능 개선 대상선정  
select rownum cnt,  
         t1.*  
from (  
         select t1.parsing_schema_name,  
                  t1.module,  
                  t1.sql_id,  
                  t1.hash_value,  
                  t1.substr_sqltext,  
                  t1.executions,  
                  t1.buffer_gets,  
                  t1.disk_reads,  
                  t1.rows_processed,  
                  t1.lio,  
                  t1.elapsed_sec,  
                  t1.cpu_sec,  
                  round(t1.cpu_time/t1.cpu_time_total*100,1) ratio_cpu,  
                  round(t1.elapsed_time/elapsed_time_total*100,1) ratio_elapsed  
        from (  
                 select parsing_schema_name,  
                          module,  
                          sql_id,  
                          hash_value,  
                          substr(sql_text,1,100) substr_sqltext,  
                          executions,  
                          buffer_gets,  
                          disk_reads,  
                          rows_processed,  
                          cpu_time,  
                          elapsed_time,  
                          round(buffer_gets/decode(executions,0,1,executions),1) lio,  
                          round(elapsed_time/decode(executions,0,1,executions)/1000000,1) elapsed_sec,  
                          round(cpu_time/decode(executions,0,1,executions)/1000000,1) cpu_sec,  
                          sum(cpu_time) over() cpu_time_total,  
                          sum(elapsed_time) over() elapsed_time_total  
                  from v$sqlarea s  
                  ) t1  
        where t1.executions >0  
        and    t1.parsing_schema_name not in ('SYS','SYSTEM','SYSMAN')  
        order by ratio_cpu DESC          
        ) t1  
where rownum <=100

-- DBA_HIST_SQLSTAT  활용 스크립트  
select sql_id,  
         schema_name,  
         module,  
         ela_ratio,  
         ela_tot,  
         cpu_ratio,  
         cpu_tot,  
         exec_ratio,  
         exec_tot,  
         lio_ratio,  
         lio_tot,  
         pio_ratio,  
         pio_tot,  
         rows_ratio,  
         rows_tot  
from (  
         select sql_id,  
                  parsing_schema_name schema_name,  
                  nvl(substr(b.module,1,15),'-') module,  
                  round(ratio_to_report(sum(b.elapsed_time_delta)) over()*100,1) as ela_ratio,  
                  round(sum(b.elapsed_time_delta)/1000000,0) as ela_tot,  
                  round(ratio_to_report(sum(b.cpu_time_delta))over()*100,1) as cpu_ratio,  
                  round(sum(b.cpu_time_delta)/1000000,0) as cpu_tot,  
                  round(ratio_to_report(sum(b.executions_delta))over()*100,1) as exec_ratio,  
                  sum(b.executions_delta) as exec_tot,  
                  round(ratio_to_report(sum(b.buffer_gets_delta))over()*100,1) as lio_ratio,  
                  sum(b.buffer_gets_delta) as lio_tot,  
                  round(ratio_to_report(sum(b.disk_reads_delta))over()*100,1) as pio_ratio,  
                  sum(b.disk_reads_delta) as pio_tot,  
                  round(ratio_to_report(sum(b.rows_processed_delta))over()*100,1) as rows_ratio,  
                  sum(b.rows_processed_delta)as rows_tot  
         from dba_hist_snapshot a,  
                 dba_hist_sqlstat b  
         where a.instance_number=1  
         and     a.begin_interval_time >= to_date(:b1,'YYYY-MM-DD')  
         and     a.end_interval_time <= to_date(:b2, 'YYYY-MM-DD') +0.99999  
         and     a.dbid=b.dbid  
         and     b.parsing_schema_name not in ('SYS','SYSTEM','SYSMAN')  
         and     a.instance_number = b.instance_number  
         and     a.snap_id=b.snap_id  
         group by b.sql_id,  
                      b.parsing_schema_name,  
                      b.module  
         order by cpu_ratio desc  
         )  
where rownum <=10;           
           
-- full table 관련 성능 개선 대상 선정

select rownum cnt,  
         t1.*  
from (                          
         select t1.parsing_schema_name,  
                  t1.module,  
                  t1.sql_id,  
                  t1.hash_value,  
                  t1.substr_sqltext,  
                  t1.executions,  
                  t1.buffer_gets,  
                  t1.disk_reads,  
                  t1.rows_processed,  
                  t1.lio,  
                  t1.elapsed_sec,  
                  t1.cpu_sec,  
                  round(t1.cpu_time/t1.cpu_time_total*100,1) ratio_cpu,  
                  round(t1.elapsed_time/elapsed_time_total*100,1) ratio_elapsed  
        from (  
                 select s.parsing_schema_name,  
                          s.module,  
                          s.sql_id,  
                          s.hash_value,  
                          s.address,  
                          substr(s.sql_text,1,100) substr_sqltext,  
                          s.executions,  
                          s.buffer_gets,  
                          s.disk_reads,  
                          s.rows_processed,  
                          s.cpu_time,  
                          s.elapsed_time,  
                          round(s.buffer_gets/decode(s.executions,0,1,executions),1) lio,  
                          round(s.elapsed_time/decode(s.executions,0,1,executions)/1000000,1) elapsed_sec,  
                          round(s.cpu_time/decode(s.executions,0,1,executions)/1000000,1) cpu_sec,  
                          sum(s.cpu_time) over() cpu_time_total,  
                          sum(s.elapsed_time) over() elapsed_time_total  
               from v$sqlarea s  
               ) t1,  
               (  
                select distinct hash_value,  
                         address  
                from v$sql_plan  
                where operation = 'TABLE ACESS'  
                and options = 'FULL'  
                ) x  
        where t1.executions > 0  
        and x.hash_value = t1.hash_value  
        and x.address = t1.address  
        and t1.parsing_schema_name not in ('SYS','SYSTEM','SYSMAN')  
        order by ratio_cpu DESC  
        ) t1  
where ROWNUM <=10        

  
-- Literal SQL 관련 성능개선 대상선정

select rownum rno,  
         t1.*  
from (  
        select max(substr_sqltext) sql_text,  
                 max(parsing_schema_name) parsing_schema_name,  
                 max(module) module,  
                 max(s.sql_id) sql_id,  
                 count(s.exact_matching_signature) literal_sql_cnt,  
                 round(sum(buffer_gets)/sum(s.executions),2) buffer_avg,  
                 round(sum(elapsed_time)/sum(s.executions),2) elapsed_avg,  
                 round(sum(rows_processed)/sum(s.executions),2) rows_processed,  
                 sum(s.executions) executions,  
                 round(sum(cpu_time)/max(cpu_time_total)*100,2) ratio_cpu,  
                 round(sum(elapsed_time)/max(elapsed_time_total)*100,2) elapsed_cpu,  
                 count(distinct s.plan_hash_value) plan_cnt  
        from (                   
                 select s.parsing_schema_name,  
                          s.module,  
                          s.sql_id,  
                          s.hash_value,  
                          s.plan_hash_value,  
                          s.address,  
                          substr(s.sql_text,1,100) substr_sqltext,  
                          s.executions,  
                          s.buffer_gets,  
                          s.disk_reads,  
                          s.rows_processed,  
                          s.cpu_time,  
                          s.elapsed_time,  
                          s.force_matching_signature,  
                          s.exact_matching_signature,  
                          round(s.buffer_gets/decode(s.executions,0,1,executions),1) lio,  
                          round(s.elapsed_time/decode(s.executions,0,1,executions)/1000000,1) elapsed_sec,  
                          round(s.cpu_time/decode(s.executions,0,1,executions)/1000000,1) cpu_sec,  
                          sum(s.cpu_time) over() cpu_time_total,  
                          sum(s.elapsed_time) over() elapsed_time_total  
                from v$sqlarea s  
                ) s  
         where s.executions > 0  
         and s.force_matching_signature <> exact_matching_signature  
         and s.parsing_schema_name not in ('SYS','SYSTEM','SYSMAN')  
         group by s.force_matching_signature  
         having count(s.exact_matching_signature) >=2  
         order by ratio_cpu DESC  
         ) t1  
where rownum <=10                                                                           

-- 배치프로그램 관련 성능 개선 대상선정  
-- 인라인뷰 o 부분에 배치 프로그램명을 입력하면 해당 프로그램에서 수행하는 모든 sql에 대한 i/o 발생량,  추출 row수, 응답시간, 수행횟수 등의 정보가 추출된다. 이 정보들을 가지고 수행시간이 오래 소요되는 부분의 sql을 추출하여 개선하면 된다.

-- 특정 program 내에서 수행된 sql 추출 스크립트

select o.object_name,  
         s.parsing_schema_name as schema,  
         s.module,  
         s.sql_id,  
         s.hash_value,  
         substr(s.sql_text,1,100) as sqltext,  
         s.executions,  
         s.buffer_gets,  
         s.disk_reads,  
         round(s.rows_processed/s.executions,1) as "Rows",  
         round(s.buffer_gets/s.executions,1) lio,  
         round(s.elapsed_time/s.executions/1000000,1) elapsed_sec,  
         round(s.cpu_time/s.executions/1000000,1) as cpu_sec,  
         round(s.elapsed_time/1000000, 1) as elapsed_time  
from ( select object_id, object_name  
          from dba_objects  
          where object_name = :Procedure_Name) o,  
          v$sqlarea s  
where o.object_id=s.program_id  
order by 14 desc                     
         

-- 1. 배치 프로그램명으로 OBJECT_ID 추출하기  
SELECT OBJECT_NAME, OBJECT_ID  
FROM DBA_OBJECTS  
WHERE OBJECT_NAME IN ('PLSQL_BATH_1','PLSQL_BATCH2'); --배치 프로그램명 입력

  
-- 2. DBA_OBJECTS에서 추출된 OBJECT_ID 값으로 V$SQLAREA의 PROGRAM_ID와 연결

col substr_text for a30  
col module for a15

select substr(sql_text,1,30) substr_text, module, program_id  
from v$sqlarea  
where program_id in ();

-- 3.sql내역확인

select t1.module, t1.substr_sqltext, t1.executions, t1.buffer_gets  
from (  
         select parsing_schema_name Schema,  
                  module,  
                  sql_id,  
                  hash_value,  
                  substr(sql_text,1,37) substr_sqltext,  
                  executions,  
                  buffer_gets,  
                  disk_reads,  
                  rows_processed,  
                  round(buffer_gets/executions,1) lio,  
                  round(elapsed_time/executions/1000000,1) elapsed_sec,  
                  round(cpu_time/executions/1000000,1) cpu_sec  
        from v$sqlarea s  
        where s.program_id in ( select object_id  
                                            from dba_objects  
                                            where object_name in (''))  
       order by 7 desc  
       )t1  
where rownum <=10;

-- 최근가장 많이 수행 된 sql과 수행점유율(수행횟수) ash  
select sql_id,  
         count(*),  
         count(*)*100/sum(count(*)) over() ratio  
from v$active_session_history  
where sample_time >= to_date(:from_time,'yyyymmdd hh24miss')  
and sample_time < to_date(:to_time,'yyyymmdd hh24miss')  
group by sql_id  
order by count(*) desc;

  
-- 특정 session이 가장 많이 수행 한 sql과 수행 점유율(수행횟수)

select sql_id,  
         count(*),  
         count(*)*100/sum(count(*)) over() ratio  
from v$active_session_history  
where sample_time >= to_date(:from_time,'yyyymmdd hh24miss')  
and sample_time < to_date(:to_time,'yyyymmdd hh24miss')  
and sessiond_id = :b1  
group by sql_id  
order by count(*) desc;

  
-- 특정 구간 이벤트 별 대기 시간  
select nvl(a.event,'ON CPU') as event,  
         count(*) as total_wait_time  
from v$active_session_history a  
where sample_time >= to_date(:from_time,'yyyymmdd hh24miss')  
and sample_time < to_date(:to_time,'yyyymmdd hh24miss')  
group by a.event  
order by total_wait_time DESC;

-- 특정 구간 cpu점유율 순 - top sql  
select ash.sql_id,  
         sum(decode(ash.session_state,'ON CPU',1,0)) "CPU",  
         sum(decode(ash.session_state,'WAITING',1,0)) -  
         sum(decode(ash.session_state,'WAITING', decode(en.wait_class,'USER I/O',1,0),0)) "Wait",  
         sum(decode(ash.session_state,'WAITING', decode(en.wait_class,'USER I/O',1,0),0)) "IO",  
         sum(decode(ash.session_state,'ON CPU',1,1)) "TOTAL"  
from v$active_session_history ash,  
        v$event_name en  
where sql_id IS NOT NULL  
and en.event#=ash.event#  
and ash.sample_time >= to_date(:from_time,'yyyymmdd hh24miss')  
and ash.sample_time < to_date(:to_time,'yyyymmdd hh24miss')  
group by sql_id  
order by sum(decode(session_state,'ON CPU',1,1)) desc;

  
-- 특정 구간 cpu 점유율 순 top session  
select ash.session_id,  
         ash.session_serial#,  
         ash.user_id,  
         ash.program,  
         sum(decode(ash.session_state, 'ON CPU',1,0)) "CPU",  
         sum(decode(ash.session_state, 'WAITING',1,0)) -  
         sum(decode(ash.session_state,'WAITING', decode(en.wait_class,'USER I/O',1,0),0)) "WAITING",  
         sum(decode(ash.session_state, 'WAITING', decode(en.wait_class,'USER I/O',1,0),0)) "IO",  
         sum(decode(session_state, 'ON CPU',1,1)) "TOTAL"  
from v$active_session_history ash,  
       v$event_name en  
where en.event# = ash.event#  
and ash.sample_time >= to_date(:from_time,'yyyymmdd hh24miss')  
and ash.sample_time < to_date(:to_time,'yyyymmdd hh24miss')  
group by session_id,  
             user_id,  
             session_serial#,  
             program  
order by sum(decode(session_state,'ON CPU',1,1)) DESC;

출처: [https://boeok.tistory.com/entry/성능-개선-대상-sql-찾기-튜닝의시작-튜닝방법론](https://boeok.tistory.com/entry/%EC%84%B1%EB%8A%A5-%EA%B0%9C%EC%84%A0-%EB%8C%80%EC%83%81-sql-%EC%B0%BE%EA%B8%B0-%ED%8A%9C%EB%8B%9D%EC%9D%98%EC%8B%9C%EC%9E%91-%ED%8A%9C%EB%8B%9D%EB%B0%A9%EB%B2%95%EB%A1%A0) [Secret:티스토리]
```