# Log Files

ConductR uses Akka logging through `akka.event.slf4j.Slf4jLoggingFilter`, which in turns uses SLF4J with Logback.

## ConductR core logging configuration

By default the log files are generated into the `logs` directory within the ConductR core installation directory. If ConductR core is installed using `.deb` and `.rpm` package, the symlink from `/var/logs/conductr` will be created to the `logs` directory.

The log rotation policy is applied to the generated log files. The rotated log files are archived in the `.gz` format, with the maximum size of `3MB` for each file. There are `14` rotated log files to be kept with the maximum total size of `60MB`.

```
<configuration>

    <appender name="console" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%date{"yyyy-MM-dd'T'HH:mm:ss'Z'",UTC} ${HOSTNAME} %-5level %logger{0} [%mdc] - %msg%n</pattern>
        </encoder>
    </appender>

    <appender name="file" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${log-file:-logs/conductr.log}</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <!--
            The log file size from Prod log for 1 week with debug enabled is about 20MB, which translates to about 3MB per day.
            -->
            <fileNamePattern>${log-file-archive:-logs/conductr-%d{yyyy-MM-dd}.%i.log.gz}</fileNamePattern>
            <maxFileSize>3MB</maxFileSize>
            <maxHistory>14</maxHistory>
            <totalSizeCap>60MB</totalSizeCap>
        </rollingPolicy>
        <encoder>
            <pattern>%date{"yyyy-MM-dd'T'HH:mm:ss'Z'",UTC} ${HOSTNAME} %-5level %logger{0} [%mdc] - %msg%n</pattern>
        </encoder>
    </appender>

    <logger name="com.typesafe" level="debug" additivity="false">
        <appender-ref ref="console"/>
        <appender-ref ref="file"/>
    </logger>

    <logger name="com.lightbend" level="debug" additivity="false">
        <appender-ref ref="console"/>
        <appender-ref ref="file"/>
    </logger>

    <logger name="org.apache.curator" level="off" additivity="false">
    </logger>

    <logger name="org.apache.zookeeper" level="off" additivity="false">
    </logger>

    <logger name="Remoting" level="off" additivity="false">
    </logger>

    <logger name="akka.remote.EndpointWriter" level="off" additivity="false">
    </logger>

    <root level="${root.loglevel:-info}">
        <appender-ref ref="console"/>
        <appender-ref ref="file"/>
    </root>

</configuration>
```

## ConductR agent logging configuration

By default the log files are generated into the `logs` directory within the ConductR agent installation directory. If ConductR agent is installed using `.deb` and `.rpm` package, the symlink from `/var/logs/conductr-agent` will be created to the `logs` directory.

The log rotation policy is applied to the generated log files. The rotated log files are archived in the `.gz` format, with the maximum size of `3MB` for each file. There are `14` rotated log files to be kept with the maximum total size of `60MB`.


```
<configuration>

    <appender name="console" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%date{"yyyy-MM-dd'T'HH:mm:ss'Z'",UTC} ${HOSTNAME} %-5level %logger{0} [%mdc] - %msg%n</pattern>
        </encoder>
    </appender>

    <appender name="file" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${log-file:-logs/conductr-agent.log}</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <!--
            The log file size from Prod log for 1 week with debug enabled is about 126MB, which translates to about 18MB per day.
            -->
            <fileNamePattern>${log-file-archive:-logs/conductr-agent-%d{yyyy-MM-dd}.%i.log.gz}</fileNamePattern>
            <maxFileSize>20MB</maxFileSize>
            <maxHistory>14</maxHistory>
            <totalSizeCap>300MB</totalSizeCap>
        </rollingPolicy>
        <encoder>
            <pattern>%date{"yyyy-MM-dd'T'HH:mm:ss'Z'",UTC} ${HOSTNAME} %-5level %logger{0} [%mdc] - %msg%n</pattern>
        </encoder>
    </appender>

    <logger name="com.typesafe" level="debug" additivity="false">
        <appender-ref ref="console"/>
        <appender-ref ref="file"/>
    </logger>

    <logger name="com.lightbend" level="debug" additivity="false">
        <appender-ref ref="console"/>
        <appender-ref ref="file"/>
    </logger>

    <logger name="Remoting" level="off" additivity="false">
    </logger>

    <logger name="akka.remote.EndpointWriter" level="off" additivity="false">
    </logger>

    <root level="${root.loglevel:-info}">
        <appender-ref ref="console"/>
        <appender-ref ref="file"/>
    </root>

</configuration>
```

## Modifying ConductR core log file locations

To modify the ConductR core log location, specify the following system property in the `/usr/share/conductr/conf/conductr.ini`.

```
-Dlog-file=/data/logs/conductr.log
```

Note that the actual path to the log file must be specified as the value to the system property.

In the example above, the `conductr.log` will be written to `/data/logs` directory. Replace this directory with your preferred directory accordingly.

To modify the log archive directory, specify the following system property in the `/usr/share/conductr/conf/conductr.ini`.

```
-Dlog-file-archive=/data/log-archives/conductr-%d{yyyy-MM-dd}.%i.log.gz
```

Note that the pattern to form the actual path to the archive file must be specified. For further details of the supported patterns, refer to the Logback documentation for `ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy`.

In the example above, the archived logs will be written to `/data/log-archives` directory. Replace this directory with your preferred directory accordingly.


## Modifying ConductR agent log file locations

To modify the ConductR agent log location, specify the following system property in the `/usr/share/conductr-agent/conf/conductr-agent.ini`.

```
-Dlog-file=/data/logs/conductr-agent.log
```

Note that the actual path to the log file must be specified as the value to the system property.

In the example above, the `conductr-agent.log` will be written to `/data/logs` directory. Replace this directory with your preferred directory accordingly.

To modify the log archive directory, specify the following system property in the `/usr/share/conductr-agent/conf/conductr-agent.ini`.

```
-Dlog-file-archive=/data/log-archives/conductr-agent-%d{yyyy-MM-dd}.%i.log.gz
```

Note that the pattern to form the actual path to the archive file must be specified. For further details of the supported patterns, refer to the Logback documentation for `ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy`.

In the example above, the archived logs will be written to `/data/log-archives` directory. Replace this directory with your preferred directory accordingly.
