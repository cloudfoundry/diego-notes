A version of the influxdb-firehose-nozzle that works with current loggregator
can be found at: https://github.com/luan/influxdb-firehose-nozzle. That fork
updates loggregator libraries and code with most recent datadog-firehose-nozzle

The nozzle could be made into a bosh release or deployed as a CF app, problem
with CF app is that it can interfere with performance tests.

Grafana (http://grafana.org) can be either deployed with bosh
(https://github.com/vito/grafana-boshrelease) or ran locally and pointed to a
remote InfluxDB.

InfluxDB can be deployed via bosh
(https://github.com/vito/influxdb-boshrelease) its configuration is really
minimal and should straightforward.

There is sample dashboard for Grafana in the same directory as this document as
`grafana-sample.json`.
