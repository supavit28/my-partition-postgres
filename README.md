# Postgres Partitioning

- [Postgres Partitioning](#postgres-partitioning)
  - [FAQ](#faq)
  - [Sample](#sample)
    - [Create table with partition](#create-table-with-partition)
    - [Create default partition](#create-default-partition)
    - [List all partition names and their parent table](#list-all-partition-names-and-their-parent-table)
    - [Query but also show its partition](#query-but-also-show-its-partition)
    - [Function to automatically create partition for next week](#function-to-automatically-create-partition-for-next-week)
    - [Create partition for next week](#create-partition-for-next-week)
    - [Attach existing table as partition for table](#attach-existing-table-as-partition-for-table)
    - [Show all partition ranges](#show-all-partition-ranges)
    - [List all pg\_cron jobs](#list-all-pg_cron-jobs)
    - [List all pg\_cron jobs' log (detail)](#list-all-pg_cron-jobs-log-detail)
    - [List all functions](#list-all-functions)
    - [List all triggers](#list-all-triggers)
    - [Drop trigger](#drop-trigger)
    - [Create trigger on database](#create-trigger-on-database)
    - [Create trigger function](#create-trigger-function)
    - [Schedule / unschedule cron](#schedule--unschedule-cron)
    - [Function to update cron job by using configuration (store in other table)](#function-to-update-cron-job-by-using-configuration-store-in-other-table)


## FAQ

1. You can't alter table to create partition on existing table.

   You need create new table and migrate data to new table with partition.

2. If table have partition key, the partition range should be part of primary key.

3. You can create table partition on table that doesn't have primary key.

4. Alembic don't allow to create migration file on table without partition key.

5. Every table that has partition should have default partition.

   Since if you want to insert data to table that can't find any partition for it, it will throw error.

   But if you have default partition, that data will be inserted to default partition.

   - Creating new partition on existing table, the existing data will not automatically move to partition.
     You need migrate data to new partition by yourself.

6. If parent table has index, partition table will automatically create index on partition table.

7. Partition range contains `FROM` and `TO`

   - **FROM** represent more than or equal to (>=)
   - **TO** represent less than (<)

   That means that if partition range is **FROM** ('2024-01-01 00:00:00') **TO** ('2024-02-01 00:00:00'),

   If inserting data is **'2024-02-01 00:00:00'**, it will be inserted to default partition not in the partition range.

   But if inserting data is **'2024-01-01 00:00:00'**, it will be inserted to the partition range.

## Sample

### Create table with partition

```sql
CREATE TABLE parking_lot_logs_new (
   created_at timestamp DEFAULT CURRENT_TIMESTAMP(0) NOT NULL,
   id uuid NOT NULL DEFAULT gen_random_uuid(),
   ticket_id varchar NULL,
   license_number varchar NOT NULL,
   license_province varchar NULL,
   mall_slug varchar NULL,
   direction varchar NOT NULL,
   client_timestamp timestamp NULL,
   gate_no varchar NULL,
   CONSTRAINT pk_parking_lot_logs PRIMARY KEY (ticket_id, client_timestamp)
) PARTITION BY RANGE (client_timestamp);
```

### Create default partition

```sql
CREATE TABLE parking_lot_logs_new_default PARTITION OF parking_lot_logs_new DEFAULT;
```

### List all partition names and their parent table

```sql
SELECT
   inhrelid::regclass AS partition_name,
   inhparent::regclass AS parent_table
FROM pg_inherits
ORDER BY parent_table, partition_name;
```

### Query but also show its partition

```sql
SELECT tableoid::regclass AS partition_name, *
FROM parking_lot_logs_new;
```

### Function to automatically create partition for next week

Timezone of database is UTC but partition should be in ICT(GMT+7)

```sql
CREATE FUNCTION create_weekly_partition() RETURNS void AS $$
   DECLARE
      start_date_char TIMESTAMP;
      start_date TIMESTAMP;
      end_date TIMESTAMP;
      partition_name TEXT;
   BEGIN
         start_date_char := date_trunc('week', now()) + interval '1 weeks';
         start_date := start_date_char - interval '7 hours';
         end_date := start_date + interval '1 week';
         partition_name := 'parking_lot_logs_new_' || to_char(start_date_char, 'YYYYMMDD');

         EXECUTE format('
            CREATE TABLE %I PARTITION OF parking_lot_logs_new
            FOR VALUES FROM (%L) TO (%L);
         ', partition_name, start_date, end_date);
   END;
$$ LANGUAGE plpgsql;
```

### Create partition for next week

Timezone of database is UTC but partition should be in ICT(GMT+7)

```sql
DO $$
DECLARE
   start_date_char TIMESTAMP;
   start_date TIMESTAMPTZ;
   end_date TIMESTAMPTZ;
   partition_name TEXT;
BEGIN
   start_date_char := date_trunc('week', now()) + interval '1 weeks';
   start_date := start_date_char - interval '7 hours';
   end_date := start_date + interval '1 week';
   partition_name := 'parking_lot_logs_new_' || to_char(start_date, 'YYYYMMDD');

   EXECUTE format('
      CREATE TABLE %I PARTITION OF parking_lot_logs_new
      FOR VALUES FROM (%L) TO (%L);
   ', partition_name, start_date, end_date);
END;
$$;
```

### Attach existing table as partition for table

```sql
ALTER TABLE parking_lot_logs_new ATTACH PARTITION parking_lot_logs_new_2 FOR
VALUES
FROM
('2024-10-10 00:00:00') TO ('2024-10-17 00:00:00');
```

### Show all partition ranges

```sql
SELECT
   relname AS partition_table,
   pg_get_expr(relpartbound,
   oid) AS partition_range
FROM
   pg_class
WHERE
   relispartition
   AND relkind = 'r';
```

### List all pg_cron jobs

```sql
SELECT * FROM cron.job;
```

### List all pg_cron jobs' log (detail)

```sql
SELECT * FROM cron.job_run_details;
```

### List all functions

```sql
SELECT
   r.routine_schema,
   r.routine_name,
   r.data_type,
   p.prosrc as function_definition
FROM information_schema.routines r
JOIN pg_namespace n ON n.nspname = r.routine_schema
JOIN pg_proc p ON p.proname = r.routine_name
   AND p.pronamespace = n.oid
WHERE r.routine_type = 'FUNCTION'
AND r.specific_schema NOT IN ('pg_catalog', 'information_schema');
```

### List all triggers

```sql
SELECT trigger_schema, trigger_name, event_object_table, event_manipulation, action_statement
FROM information_schema.triggers;
```

### Drop trigger

```sql
DROP TRIGGER config_update_trigger ON db_config;
```

### Create trigger on database
```sql
CREATE TRIGGER config_update_trigger
AFTER INSERT OR UPDATE OR DELETE ON db_config
FOR EACH ROW
EXECUTE FUNCTION trigger_parking_lot_logs_cron();
```

### Create trigger function
```sql
CREATE FUNCTION trigger_parking_lot_logs_cron() RETURNS TRIGGER AS $$
BEGIN
   IF NEW.cfg_key = 'cron_time_parking_lot_log_weekly_partition' OR OLD.cfg_key = 'cron_time_parking_lot_log_weekly_partition' THEN
      PERFORM update_cron_job();
   END IF;
   RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

### Schedule / unschedule cron

```sql
-- Set cron schedule
SELECT cron.schedule('create_weekly_parking_lot_logs_partition', '30 10 * * *', 'SELECT create_weekly_partition();');
-- Unschedule cron job
SELECT cron.unschedule('create_weekly_parking_lot_logs_partition');
```

### Function to update cron job by using configuration (store in other table)

Sample data in db_config table

id|cfg_key |cfg_val |
--|------------------------------------------|-----------|
1|cron_time_parking_lot_log_weekly_partition|10 05 \* \* \*|

Sample script

```sql
CREATE FUNCTION update_cron_job() RETURNS void AS $$
   DECLARE
      cron_schedule text;
      cron_name text;
   BEGIN
      -- Get the cron schedule from db_config table
      SELECT cfg_val INTO cron_schedule
      FROM db_config
      WHERE cfg_key = 'cron_time_parking_lot_log_weekly_partition' LIMIT 1;

      -- If the schedule exists, create the cron job
      IF FOUND THEN

         select jobname into cron_name from cron.job where jobname = 'create_weekly_parking_lot_logs_partition';
         if cron_name is not null then
            -- Delete existing job with the same name if it exists (UPDATE triggered)
            PERFORM cron.unschedule(cron_name);
            RAISE NOTICE 'Unschedule old cron job';
         end if;

         -- Create the new job with the schedule from the database (INSERT triggered)
         PERFORM cron.schedule(
            'create_weekly_parking_lot_logs_partition',
            cron_schedule,
            'SELECT create_weekly_partition();'
         );
         RAISE NOTICE 'Scheduled job with cron pattern: %', cron_schedule;

      -- If can't found schedule, unschedule job (use when DELETE is triggered)
      ELSE
         PERFORM cron.unschedule('create_weekly_parking_lot_logs_partition');
         RAISE EXCEPTION 'Cron schedule not found in db_config';
      END IF;
   END;
$$ LANGUAGE plpgsql;
```
