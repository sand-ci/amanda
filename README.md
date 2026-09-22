# perfSONAR collector to run at UC

[![Build PerfSONAR Logstash dockerhub image](https://github.com/sand-ci/amanda/actions/workflows/ps-collector.yaml/badge.svg)](https://github.com/sand-ci/amanda/actions/workflows/ps-collector.yaml)

[![Build pS Mapper dockerhub image](https://github.com/sand-ci/amanda/actions/workflows/ps-mapper.yaml/badge.svg)](https://github.com/sand-ci/amanda/actions/workflows/ps-mapper.yaml)

To get logstash statistics do:
curl -XGET 'localhost:9600/_node/stats/pipelines/ps-collector?pretty'

and

curl -XGET 'localhost:9600/_node/stats/os?pretty'
curl -XGET 'localhost:9600/_node/hot_threads?pretty'

To check memcached status:

yum install telnet
telnet memcached 11211
stats items

Before starting logstash, make sure that the ps-mapper job at least once finished fine.
One can manually execute it: kubectl create job --from=cronjob/<name of cronjob> <name of job>

To manually flush the cache:
echo 'flush_all' | nc memcached.collectors.svc.cluster.local 11211

## mapper

`mapping.py` builds the lookup data that logstash uses to enrich perfSONAR events, and caches it in memcached:

- Loads production PWA perfSONAR hosts via `psconfig.api`.
- Loads site network routes from WLCG CRIC and builds an IP-subnet-to-netsite mapping.
- Loads active PerfSonar services from WLCG CRIC, resolves each host's IP, and for every host determines its VO(s), site name(s), rcsite, netsite, flavor, and production status.
- Loads the SciTags experiment/activity mapping from scitags.org.
- Writes all of this (`vo_<host>`, `sitename_<host>`, `rcsite_<host>`, `netsite_<host>`, `production_<host>`, `exp_<id>`, `exp_<id>_act_<id>`, etc.) into memcached as key/value pairs, keyed by hostname or experiment/activity id.

It runs as a Kubernetes CronJob and must complete successfully at least once before logstash starts, since the ps-collector pipeline relies on this cached mapping data to enrich incoming perfSONAR measurements.

