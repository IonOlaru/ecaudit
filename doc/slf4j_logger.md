# SLF4J Logger

The SLF4J logger can be configured with Logback to produce various log file formats,
including clear text log files.

The SLF4J logger is used by default.
This can be configured explicitly with settings in the ```audit.yaml``` as follows.

```YAML
logger_backend:
    - class_name: com.ericsson.bss.cassandra.ecaudit.logger.Slf4jAuditLogger
```

In the following sections you'll find examples on how to configure the SLF4J logger further.


## Custom Log Message Format

To configure a custom log message format the following parameters can be configured in the ```audit.yaml``` file:

| Parameter   | Description                                                       | Default |
| ----------- | ----------------------------------------------------------------- | --------------- |
| log_format  | Parameterized log message formatting string, see examples below  | the "legacy" format, see [README](../README.md) |
| time_format | time formatter pattern, see examples below or [DateTimeFormatter](https://docs.oracle.com/javase/8/docs/api/java/time/format/DateTimeFormatter.html#patterns) | number of millis since EPOCH |
| time_zone   | the time zone id, see examples below or [ZoneId](https://docs.oracle.com/javase/8/docs/api/java/time/ZoneId.html#of-java.lang.String-) | system default |

It is possible to configure a parameterized log message by providing a formatting string.
The following fields are available:

| Field       | Field Value                                                       | Mandatory Field |
| ----------- | ----------------------------------------------------------------- | --------------- |
| CLIENT      | Client IP address                                                 | yes             |
| USER        | Username of the authenticated user                                | yes             |
| BATCH_ID    | Internal identifier shared by all statements in a batch operation | no              |
| STATUS      | Value is either ATTEMPT or FAILED                                 | yes             |
| OPERATION   | The CQL statement or a textual description of the operation       | yes             |
| TIMESTAMP   | The system timestamp of the request (\*) (**)                     | yes             |
| COORDINATOR | Coordinator IP address (host address)                             | yes             |

(*) - This timestamp is more accurate than the Logback time (since that is written asynchronously).
If this timestamp is used, then the Logback timestamp can be removed by reconfiguring the encoder pattern in logback.xml.

(**) - It is possible to configure a custom display format.

Modify the ```audit.yaml``` configuration file.
Field name goes between ```${``` and ```}``` (*bash*-style parameter substitution).
Use the example below as a template to define the log message format.

```YAML
logger_backend:
    - class_name: com.ericsson.bss.cassandra.ecaudit.logger.Slf4jAuditLogger
      parameters:
      - log_format: "client=${CLIENT}, user=${USER}, status=${STATUS}, operation='${OPERATION}'"
```

Which will generate logs entries like this:

```
15:18:14.089 - client=127.0.0.1, user=cassandra, status=ATTEMPT, operation='SELECT * FROM students'
```

*Conditional formatting* of fields is also available, which makes it possible to only log the field value and
its descriptive text only if a value exists, which can be useful for optional fields like BATCH_ID.
Conditional fields and its associated text goes between ```{?``` and ```?}```.
The example below use conditional formatting for the batch id to get different log messges depending on if the operation
was part of a batch or not. Also the TIMESTAMP field have a custom time format configured.

```YAML
logger_backend:
    - class_name: com.ericsson.bss.cassandra.ecaudit.logger.Slf4jAuditLogger
      parameters:
      - log_format: "${TIMESTAMP}-> client=${CLIENT}, user=${USER}, status=${STATUS}, {?batch-id=${BATCH_ID}, ?}operation='${OPERATION}'"
        time_format: "yyyy-MM-dd HH:mm:ss.SSS"
        time_zone: UTC
```

Which will generate logs entries like this (assuming Logback pattern does not contain timestamp as well):

```
2019-02-28 15:18:14.089-> client=127.0.0.1, user=cassandra, status=ATTEMPT, operation='SELECT * FROM students'
2019-02-28 15:18:14.090-> client=127.0.0.1, user=cassandra, status=ATTEMPT, batch-id=6f3cae9b-f1f1-4a4c-baa2-ed168ee79f9d, operation='INSERT INTO ecks.ectbl (partk, clustk, value) VALUES (?, ?, ?)[1, '1', 'valid']'
2019-02-28 15:18:14.091-> client=127.0.0.1, user=cassandra, status=ATTEMPT, batch-id=6f3cae9b-f1f1-4a4c-baa2-ed168ee79f9d, operation='INSERT INTO ecks.ectbl (partk, clustk, value) VALUES (?, ?, ?)[2, '2', 'valid']'
```


## Configure Logback

When using the SLF4J logger, update the Cassandra ```logback.xml``` file to define path and rolling policy
for generated audit logs.
Add an appender and logger in your ```logback.xml``` configuration.
The logger name of the audit records is ```ECAUDIT```.

Tuning tips:
* The asynchronous appender can _improve or demote_ performance depending on your setup.
* Compression on rolling files may impact performance significantly.
* If you are logging large volumes of data, make sure your storage can keep up.

In the example snippet below,
Logback is configured to use a rolling policy with a synchronous appender.
Run performance tests on your workload to find out what settings works best for you.


```XML
<!--audit log-->
<appender name="AUDIT-FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
  <file>${cassandra.logdir}/audit/audit.log</file>
  <encoder>
    <pattern>%d{HH:mm:ss.SSS} - %msg%n</pattern>
    <immediateFlush>true</immediateFlush>
  </encoder>
  <rollingPolicy class="ch.qos.logback.core.rolling.FixedWindowRollingPolicy">
    <fileNamePattern>${cassandra.logdir}/audit/audit.log.%i</fileNamePattern>
    <minIndex>1</minIndex>
    <maxIndex>5</maxIndex>
  </rollingPolicy>
  <triggeringPolicy class="ch.qos.logback.core.rolling.SizeBasedTriggeringPolicy">
    <maxFileSize>200MB</maxFileSize>
  </triggeringPolicy>
</appender>

<logger name="ECAUDIT" level="INFO" additivity="false">
  <appender-ref ref="AUDIT-FILE" />
</logger>
```

There are many ways to configure appenders with Logback.
Refer to the [official documentation](https://logback.qos.ch/manual/appenders.html) for details.

