# MySQL Query Logging

## Enable Query logging on the database

```sql
SET global general_log = 1;
SET global log_output = 'table';
```

## Query

```sql
SELECT * FROM mysql.general_log;
-- OR
SELECT event_time, user_host, thread_id, server_id, command_type, CONVERT(argument USING utf8) FROM mysql.general_log;
```

## Disable

```sql
SET global general_log = 0;
```

## Clear Log

```sql
TRUNCATE table mysql.general_log;
```
