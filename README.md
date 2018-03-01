# ELK Notes

Notee on setup for local development of systems utilizing Elastic
Search, Logstash, Kilbana.

I have been developing software on a Mac for over 15 years now and
have been though quiate a few dev stacks, from the likes of MAMP and
Vagrant/Virtualbox to hybrid development and production environments,
VPNs and network mounted file systems. Thease days Docker has been a
fantastically elegant solution for local development. Containers
have allow me to replicate fairly sophisticated production systems
quickly on my local Mac.


## Getting Started with ELK

I am going to be using the official OSS [Docker containers from
elastic.io](https://www.docker.elastic.co/).

### Product stack:

- Elastic Search
- Logstash
- Beats
- Kilbana

#### Network

Create a Docker network so we can use container names to access
local services across containers.

```bash
docker network create elknet
```

#### Elastic Search

```bash
docker run -d --name=elasticsearch \
    --net elknet \
    -p 9200:9200 \
    -p 9300:9300 \
    -e "discovery.type=single-node" \
    docker.elastic.co/elasticsearch/elasticsearch-oss:6.2.2
```

Check the status of the new Elasticsearch container with a curl call
`curl localhost:9200/` or browse to http://localhost:9200/

You should see something like the following:

```json
{
  "name": "cOp0B0a",
  "cluster_name": "docker-cluster",
  "cluster_uuid": "03cHIPx2QweVYSNhCky_IA",
  "version": {
    "number": "6.2.2",
    "build_hash": "10b1edd",
    "build_date": "2018-02-16T19:01:30.685723Z",
    "build_snapshot": false,
    "lucene_version": "7.2.1",
    "minimum_wire_compatibility_version": "5.6.0",
    "minimum_index_compatibility_version": "5.0.0"
  },
  "tagline": "You Know, for Search"
}
```

Test out the new ElasticSearch container by **setting** some simple test
data.

```bash
curl -XPUT 'localhost:9200/test/_doc/1?pretty' \
    -H 'Content-Type: application/json' -d' 
    {
        "user" : "cjimti",
        "post_date" : "2018-02-28T15:12:12",
        "message" : "Testing the local Elasticsearch"
    }'
```

**Get** the document we just added.

```bash
curl -XGET 'localhost:9200/test/_doc/1?pretty'
```

You should get back something like the following

```bash
{
  "_index" : "test",
  "_type" : "_doc",
  "_id" : "1",
  "_version" : 4,
  "found" : true,
  "_source" : {
    "user" : "cjimti",
    "post_date" : "2018-02-28T15:12:12",
    "message" : "Testing the local Elasticsearch"
  }
}
```

#### Logstash

The Logstash data flow:

`Source -> Input -> Filter -> Output -> Destination`

The default Logstash configuration takes input from
[Beats](https://www.elastic.co/products/beats). And outputs to
standard out. The following is the configuration file shipped with
the OSS 6.2.2 container.

```bash
input {
  beats {
    port => 5044
  }
}

output {
  stdout {
    codec => rubydebug
  }
}
```

Next we fire up a Logstash container in the foreground with default
configuration. This is to test sending data in from beats and watching
it output to our terminal via the standard out defined in the default
configuration.

```bash
docker run -d --name=logstash \
    --net elknet \
    -p 5044:5044 \
    docker.elastic.co/logstash/logstash-oss:6.2.2
```

We will run [Packetbeat](https://www.elastic.co/downloads/beats/packetbeat)
on our local workstation to test Lostash.

Download and install [Packetbeat 6.2.2 for Mac](https://artifacts.elastic.co/downloads/beats/packetbeat/packetbeat-6.2.2-darwin-x86_64.tar.gz).

```bash
wget https://artifacts.elastic.co/downloads/beats/packetbeat/packetbeat-6.2.2-darwin-x86_64.tar.gz

tar -xzvf packetbeat-6.2.2-darwin-x86_64.tar.gz

cd packetbeat-6.2.2-darwin-x86_64
```

Packetbeat requires it's config file be owned by the account running it.
Since we will be using sudo we will give need to give ownership to the
user **root**.

```bash
chown root packetbeat.yml

sudo ./packetbeat -e -c packetbeat.yml
```

If you fired up Packetbeat right now it would begin sending data to our
Elasticsearch running on localhost:9200. However we want to use it to test
our new Logstash container. Make the following change to configuration.

Open **packetbeat.yml** in the packetbeat-6.2.2-darwin-x86_64 folder in
a text editor.

Toward the end of the file you should see an Outputs section like the
following:

```yaml
#================================ Outputs =====================================

# Configure what output to use when sending the data collected by the beat.

#-------------------------- Elasticsearch output ------------------------------
output.elasticsearch:
  # Array of hosts to connect to.
  hosts: ["localhost:9200"]

  # Optional protocol and basic auth credentials.
  #protocol: "https"
  #username: "elastic"
  #password: "changeme"

#----------------------------- Logstash output --------------------------------
#output.logstash:
  # The Logstash hosts
  #hosts: ["localhost:5044"]

  # Optional SSL. By default is off.
  # List of root certificates for HTTPS server verifications
  #ssl.certificate_authorities: ["/etc/pki/root/ca.pem"]

  # Certificate for SSL client authentication
  #ssl.certificate: "/etc/pki/client/cert.pem"

  # Client Certificate Key
  #ssl.key: "/etc/pki/client/cert.key"
```

Comment out `output.elasticsearch:` and `hosts: ["localhost:9200"]` in
the Elasticsearch output section and uncomment
`#output.logstash:` and `#hosts: ["localhost:5044"]` in the Logstash
output section.

Run packetbeat on your local Mac with the new configuration.

```bash
sudo ./packetbeat -e -c ./packetbeat.yml
```

In another terminal tail the Logstash output: `docker logs -f logstash`

You should start to see data coming out of logstash since the output
configuration is set to standard out and looks like this:

```plain
output {
  stdout {
    codec => rubydebug
  }
}
```

Some stdout (standard out) examples of Packetbeat data should look like: this:

```plain
{
    "source" => {
        "stats" => {
            "net_packets_total" => 1,
              "net_bytes_total" => 60
        },
          "mac" => "XX:XX:XX:XX:XX:XX"
    },
    "beat" => {
        "name" => "Straylight-Workstation.local",
        "hostname" => "Straylight-Workstation.local",
        "version" => "6.2.2"
    },
    "host" => "Straylight-Workstation.local",
    "dest" => {
        "mac" => "ff:ff:ff:ff:ff:ff"
    },
    "@timestamp" => 2018-03-01T02:25:19.980Z,
       "flow_id" => "AQAA//////////////8BAAAADSjXYjn///////8",
         "final" => false,
    "start_time" => "2018-03-01T02:25:16.075Z",
          "type" => "flow",
     "last_time" => "2018-03-01T02:25:16.075Z",
      "@version" => "1",
          "tags" => [
            [0] "beats_input_raw_event"
        ]
}

```


Checkout the [command line](https://www.elastic.co/guide/en/beats/packetbeat/current/command-line-options.html) and [configuration](https://www.elastic.co/guide/en/beats/packetbeat/current/configuring-howto-packetbeat.html) documentation for Packetbeat.


See a sample Packetbeat configuration file included in this repository at [./config/packetbeat.yml](./config/packetbeat.yml)

Although we are using Packetbeat for testing our ELK stack it is a very
useful utility for network monitoring or correlating network activity
with other data for performance analysis and diagnostics.

Packetbeat works great as is own [docker container](https://www.elastic.co/guide/en/beats/packetbeat/current/running-on-docker.html). You can
optionally start one and live it running to get a good flow of stats
into our new ELK stack.

From the base of this repository run the following: (you need to be in
base in order to mount the configuration file in
`$(pwd)/conf/packetbeat.yml`.

```bash
docker run -d --name packetbeat \
    --cap-add=NET_ADMIN --network=host \
    -v $(pwd)/conf/packetbeat.yml:/usr/share/packetbeat/packetbeat.yml \
    docker.elastic.co/beats/packetbeat:6.2.2
```

#### Logstash ElasticSearch Configuration

This repository contains a Logstash configuration file in
./conf/logstash.conf. The following config sets beats as an input
and elastic search as output. We do not need filters at this point
since beats already sends our data in a useful format for Elasticsearch.

```plain
input {
  beats {
    port => 5044
  }
}

output {
    elasticsearch { hosts => ["logstash:9200"] }
}
```

Stop the current Logstash container by running `docker stop logstash`
and `docker rm logstash`.

Run the Logstash container with a new configuration defining output to
Elastic search.

```bash
docker run -d -it --name=logstash \
    --net elknet \
    -p 5044:5044 \
    -v $(pwd)/conf/logstash.conf:/usr/share/logstash/pipeline/logstash.conf \
    docker.elastic.co/logstash/logstash-oss:6.2.2
```