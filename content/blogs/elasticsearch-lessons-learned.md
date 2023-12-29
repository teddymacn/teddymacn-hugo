---
author: Teddy
title: "Elasticsearch Mantanence Lessons Learned Today"
date: "2016-06-17"
#description: ""
summary: ""
tags: ["elasticsearch"]
#ShowToc: true
#TocOpen: true
---

**The elasticsearch cluster was down!**

Today I troubleshooted an Elasticsearch-cluster-down issue. We have a 3-node Elasticsearch cluster receiving hundreds of Giga of tracking data every day. And this afternoon, it was suddenly down and all our kibana dashboards failed to load any data from it.

From [elasticsearch-kopf](https://github.com/lmenezes/elasticsearch-kopf) monitor, we could see more than half of the shards are unallocated, so it sounds like at least 2 nodes were just restarted for some reason. Coz of our cluster setting is each index has one primary and one replica, until at least the primary shards are allocated, the indices are not able to be loaded. The shards are being slowly allocated automatically. If I'm patient enough and just wait for a while, it should be recover by itself in my understanding. So I try to wait. After 10 minutes, some dashboards could display, which looks good. But after 30 minutes, from kopf, I could see the HEAP of the master node keeps increasing, and eventually full. And the entire cluster becomes no responsive again. Restart the master node, but the HEAP still keeps increasing and be full and cluster down again.

**Is it because the shards being reallocated too slow?**

Someone in my team is working one a kibana dashboard, and since it suddenly stopped working, he reached me and asked if I could help to do something to speed up the recovery. And I'm guessing. Is it because the shards were reallocated too slow? I have a script which calls the elasticsearch API to do force shard reallocation. When I created the script, it was for fixing very few shards which could not be reallocated by elasticsearch itself. It looks like below:

``` bash
IFS=$'\n'
for line in $(curl -s 'server_name/es/_cat/shards' | fgrep UNASSIGNED); do
  INDEX=$(echo $line | (awk '{print $1}'))
  SHARD=$(echo $line | (awk '{print $2}'))

  curl -XPOST 'server_name/es/_cluster/reroute' -d '{
     "commands": [
        {
            "allocate": {
                "index": "'$INDEX'",
                "shard": '$SHARD',
                "node": "ETELASTIC1",
                "allow_primary": true
          }
        }
    ]
  }'

  # try again since the first try might fail if the target node already has a copy of the shard
  curl -XPOST 'server_name/es/_cluster/reroute' -d '{
     "commands": [
        {
            "allocate": {
                "index": "'$INDEX'",
                "shard": '$SHARD',
                "node": "ETELASTIC2",
                "allow_primary": true
          }
        }
    ]
  }'

  sleep 2

done
```

Restarted the master node and executed the script, and it keeps running. It'll take some time coz I have 10K of unallocated shards. The reallocation becomes faster. But no luck that before it finishes, the HEAP became full again. So sounds like the guess is not correct.

**What causes the HEAP increasing then?**

Checked the log of elasticsearch, and it shows there are only very few small cheap reads during the cluster down & recovery time. Checked the nodes breaker API and the fielddata cache is also not big. So what else might cost elasticsearch memory? If not reads, then it might be writes?

Luckily we have centralized logstash server for parsing all different types of tracking logs and feeding them to elasticsearch servers, so I simply stopped all of them. Restarted the master node again, wait and eventually, the elasticsearch cluster recovered without HEAP issue. Great! Here it is. So it should be because we configured our logstash jobs to always try re-connect elasticsearch servers when server connection timeout or no response, and since we have many logstash jobs trying to reconnect, it costs too much elasticsearch server-side memory in HEAP.

Let's try some kibana dashboards, WAIT! Why some of the dashboards returns 404? It is impossible that people deleted them during my troubleshooting, we rarely delete dashboards. It makes me nerves. Coz it looks like reallocation of elasticsearch shards might cause data loss. WTF! If that's true, how could we truct elasticsearch anymore?

**Always human's fault!**

Ok, ask google. And, found the reason. Usually, it is always human's fault. I mean, my fault, not elasticsearch's. It is because of the ["allow_primary=true"](https://discuss.elastic.co/t/cluster-reroute-and-potential-data-loss/15573) option in my "force reallocation" script. Bascially, when all the shards of an index are unallocated yet, and if you do reroute with the "allow_primary=true" option, some shards might become empty, and you LOSS data! Luckily, it is some shards of the kibana metadata index were lost, so I realized this issue, if it is only some tracking index loss, it is much harder for me to be awared.

But I'm lucky enough, coz:

1. I have daily backup of the kibana metadata index with [elasticdump](https://github.com/taskrabbit/elasticsearch-dump), and no much dashboard changes everyday;
2. We keeps latest 3-day raw data of all the tracking logs, so I could easily re-index them;

Restored from the kibana metadata index backup. And finally, all the dashboards are back!

**Summary**

Several lessons were learned here:

1. When many elasticsearch cluster nodes are restarted, to avoid HEAP spike, better to temporarily stop all connection attempts;
2. Avoid setting allow_primary=true when reroute shards via API;
3. Don't forget backup! It could save you some day!
