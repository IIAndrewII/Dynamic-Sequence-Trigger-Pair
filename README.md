# Dynamic Sequence/Trigger Pair
Dynamic Sequence/Trigger Pair using  Oracle PL/SQL on every table in the database.

## Step:1 -  extract table names and their numeric primary key

```sql

--Project (create seq, trg pairs on all tables in the schema to auto create primary key)


set serveroutput on size 1000000

declare
max_n number(30) ;

-- tables cursor
cursor tables_cursor is 
                SELECT a.OBJECT_NAME as TABLE_NAME, nvl(min(c.column_name),0) as pk_column
                      FROM ALL_OBJECTS a , all_cons_columns c
                      WHERE a.OBJECT_NAME = c.TABLE_NAME 
                           and a.OWNER = 'HR'
                           and a.OBJECT_TYPE = 'TABLE'
                           and c.constraint_name in (SELECT constraint_name FROM all_constraints WHERE CONSTRAINT_TYPE = 'P')
                            group by(a.OBJECT_NAME) -- if there is a composite key choose only one
                            having min(c.column_name) NOT IN (SELECT column_name FROM  user_tab_columns WHERE data_type <> 'NUMBER'); 
             
```
Sample from the output of the select statment:

| TABLE_NAME | pk_column |
| :---: | :---: |
|LOCATIONS |	LOCATION_ID|
|EMPLOYEES |	EMPLOYEE_ID|
|REGIONS |	REGION_ID|
|COUNTRIES |	COUNTRY_ID|
|JOB_HISTORY |	EMPLOYEE_ID|
|DEPARTMENTS |	DEPARTMENT_ID|




```sql
-- sequence names cursor
cursor sequence_name_cursor is 
    select sequence_name from user_sequences;
```
Sample from the output of the select statment:

| sequence_name |
| :---: |
|APPLY$_DEST_OBJ_ID | 
 | APPLY$_ERROR_HANDLER_SEQUENCE | 
 | APPLY$_SOURCE_OBJ_ID | 
 | AQ$_ALERT_QT_N | 
 | AQ$_AQ$_MEM_MC_N | 
 | AQ$_AQ_PROP_TABLE_N | 
 | AQ$_CHAINSEQ | 
 | AQ$_IOTENQTXID | 
 | AQ$_KUPC$DATAPUMP_QUETAB_N | 
 | AQ$_NONDURSUB_SEQUENCE | 
 | AQ$_PROPAGATION_SEQUENCE | 
 | AQ$_PUBLISHER_SEQUENCE | 
 | AQ$_RULE_SEQUENCE | 
 | AQ$_RULE_SET_SEQUENCE | 
 | AQ$_SCHEDULER$_EVENT_QTAB_N | 
 | AQ$_SCHEDULER$_REMDB_JOBQTAB_N | 
 | AQ$_SCHEDULER_FILEWATCHER_QT_N | 
 | AQ$_SYS$SERVICE_METRICS_TAB_N | 
 | AQ$_TRANS_SEQUENCE | 


 
## Step:2 -  drop all sequences -  Create sequence for each table - Create Trigger

```sql

    
Begin

        -- drop all sequences
        for sequence_name_record in sequence_name_cursor loop
                Execute immediate '
                 drop sequence '||sequence_name_record.sequence_name;
            end loop;      
            
               
            
        for t_record in tables_cursor loop
                -- Get Maximum ID
                    EXECUTE IMMEDIATE '
                    SELECT (NVL(MAX('||t_record.pk_column||'),0)+1) 
                    FROM HR.' || t_record.table_name
                    INTO max_n;

```

## Step:3 -  Create sequence for each table

```sql

    
Begin

        -- drop all sequences
        for sequence_name_record in sequence_name_cursor loop
                Execute immediate '
                 drop sequence '||sequence_name_record.sequence_name;
            end loop;      
            
               
            
        for t_record in tables_cursor loop
                -- Get Maximum ID
                    EXECUTE IMMEDIATE '
                    SELECT (NVL(MAX('||t_record.pk_column||'),0)+1) 
                    FROM HR.' || t_record.table_name
                    INTO max_n;

                -- Create sequence for the table
                Execute immediate '
                 CREATE SEQUENCE '||t_record.table_name||'_SEQ
                START WITH '||max_n||'
                  MAXVALUE 999999999999999999999999999
                  MINVALUE 1
                  NOCYCLE
                  CACHE 20
                  NOORDER';
                  
```

## Step:4 -  Create Trigger

```sql 

Begin

        -- drop all sequences
        for sequence_name_record in sequence_name_cursor loop
                Execute immediate '
                 drop sequence '||sequence_name_record.sequence_name;
            end loop;      
            
               
            
        for t_record in tables_cursor loop
                -- Get Maximum ID
                    EXECUTE IMMEDIATE '
                    SELECT (NVL(MAX('||t_record.pk_column||'),0)+1) 
                    FROM HR.' || t_record.table_name
                    INTO max_n;

                -- Create sequence for the table
                Execute immediate '
                 CREATE SEQUENCE '||t_record.table_name||'_SEQ
                START WITH '||max_n||'
                  MAXVALUE 999999999999999999999999999
                  MINVALUE 1
                  NOCYCLE
                  CACHE 20
                  NOORDER';
-- Create Trigger
                Execute immediate'
                CREATE OR REPLACE TRIGGER '||t_record.table_name||'_tr
                BEFORE INSERT
                ON HR.'||t_record.table_name||'
                REFERENCING NEW AS New OLD AS Old
                FOR EACH ROW
                BEGIN
                  :new.'||t_record.pk_column||' := '||t_record.table_name||'_SEQ.nextval;
                        END;';
           
                        dbms_output.put_line('table_name:'||t_record.table_name||'    pk_column:'||t_record.pk_column||'   max_n:'|| max_n);
            end loop;             
end;
```
