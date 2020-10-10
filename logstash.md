# Arrancar

Podemos arrancar logstash con la configuración de ejemplo:

```ps
.\bin\logstash.bat -f .\config\logstash-sample.conf
```

Logstash tiene tres componentes:
- input
- filter
- output

En cada uno de estas etapas podemos configurar uno o varios plugins. Por ejemplo [aqui](./logstash/simple.conf) tenemos una configuración de ejemplo. Si arrancamos logstash con esta configuración:

```ps
.\bin\logstash.bat -f .\config\simple.conf

...

[2020-10-08T07:57:10,489][INFO ][logstash.agent           ] Successfully started Logstash API endpoint {:port=>9600}
hola como estas
{
    "@timestamp" => 2020-10-08T05:57:37.173Z,
          "host" => "Eugenio",
      "@version" => "1",
       "message" => "HOLA COMO ESTAS\r"
}
```

Se inclyen varios plugins con la instalación. Podemos verlos haciendo:

```ps
.\logstash-plugin.bat list

logstash-codec-avro
logstash-codec-cef
logstash-codec-collectd
logstash-codec-dots
logstash-codec-edn
logstash-codec-edn_lines
logstash-codec-es_bulk
logstash-codec-fluent
logstash-codec-graphite
logstash-codec-json
logstash-codec-json_lines
logstash-codec-line
logstash-codec-msgpack
logstash-codec-multiline
logstash-codec-netflow
logstash-codec-plain
logstash-codec-rubydebug
logstash-filter-aggregate
logstash-filter-anonymize
logstash-filter-cidr
logstash-filter-clone
logstash-filter-csv
logstash-filter-date
logstash-filter-de_dot
logstash-filter-dissect
logstash-filter-dns
logstash-filter-drop
logstash-filter-elasticsearch
logstash-filter-fingerprint
logstash-filter-geoip
logstash-filter-grok
logstash-filter-http
logstash-filter-json
logstash-filter-kv
logstash-filter-memcached
logstash-filter-metrics
logstash-filter-mutate
logstash-filter-prune
logstash-filter-ruby
logstash-filter-sleep
logstash-filter-split
logstash-filter-syslog_pri
logstash-filter-throttle
logstash-filter-translate
logstash-filter-truncate
logstash-filter-urldecode
logstash-filter-useragent
logstash-filter-uuid
logstash-filter-xml
logstash-input-azure_event_hubs
logstash-input-beats
logstash-input-couchdb_changes
logstash-input-dead_letter_queue
logstash-input-elasticsearch
logstash-input-exec
logstash-input-file
logstash-input-ganglia
logstash-input-gelf
logstash-input-generator
logstash-input-graphite
logstash-input-heartbeat
logstash-input-http
logstash-input-http_poller
logstash-input-imap
logstash-input-jms
logstash-input-pipe
logstash-input-redis
logstash-input-s3
logstash-input-snmp
logstash-input-snmptrap
logstash-input-sqs
logstash-input-stdin
logstash-input-syslog
logstash-input-tcp
logstash-input-twitter
logstash-input-udp
logstash-input-unix
logstash-integration-jdbc
 ├── logstash-input-jdbc
 ├── logstash-filter-jdbc_streaming
 └── logstash-filter-jdbc_static
logstash-integration-kafka
 ├── logstash-input-kafka
 └── logstash-output-kafka
logstash-integration-rabbitmq
 ├── logstash-input-rabbitmq
 └── logstash-output-rabbitmq
logstash-output-cloudwatch
logstash-output-csv
logstash-output-elastic_app_search
logstash-output-elasticsearch
logstash-output-email
logstash-output-file
logstash-output-graphite
logstash-output-http
logstash-output-lumberjack
logstash-output-nagios
logstash-output-null
logstash-output-pipe
logstash-output-redis
logstash-output-s3
logstash-output-sns
logstash-output-sqs
logstash-output-stdout
logstash-output-tcp
logstash-output-udp
logstash-output-webhdfs
logstash-patterns-core
```

Podemos filtrar. Por ejemplo todos los output plugins:

```ps
.\logstash-plugin.bat list --group output
```

Buscar plugins con un nombre determinado. Por ejemplo que tengan sean relativos a Kafka:

```ps
.\logstash-plugin.bat list "kafka"

logstash-integration-kafka
 ├── logstash-input-kafka
 └── logstash-output-kafka
```

## Ejemplo. Input File, Output csv y stdout

Vamos a demostrar una caso con un input de tipo file:

```ps
.\bin\logstash.bat -f .\config\simple1.conf
```

Hay otros inputs como `beats`, `jdbc`, `imap`...
