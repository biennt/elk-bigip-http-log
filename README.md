# elk-bigip-http-log
Sending log from a bigip http/https virtual server to a dockerized ELK via HSL

## Quick start
Add the following line into /etc/sysctl.conf
```
vm.max_map_count=262144
```
Then
```
sudo sysctl -p
```
Create an empty directory to store the indexes/data (eg: /elk) and run the docker image
```
docker run -d -p 5601:5601 -p 9200:9200 -p 5140:5140 -p 5140:5140/udp -v /elk:/var/lib/elasticsearch --name elk biennt/elk:bigip
```

On the BIG-IP, 
- Create a pool named elklog, pointing to the machine running ELK above, destination port is 5140
- Create an iRule and apply it to the virtual server you want to collect log. The content of the irule is below
```
when HTTP_REQUEST {
   set logString "<190>#"
   set member_ip "0.0.0.0"
   set member_port "0"
   set reqtimestamp [clock clicks -milliseconds]
   set vs_name [virtual name]
   set client_ip [IP::client_addr]
   set virtual_ip [IP::local_addr]
   set virtual_port [TCP::local_port]
   set client_port [TCP::client_port]
   set domain [HTTP::host]
   set method [HTTP::method]
   set uri [HTTP::uri]
   set agent [HTTP::header "User-Agent"]
   set logString "$logString${reqtimestamp}#${vs_name}#${client_ip}#${client_port}#${virtual_ip}#${virtual_port}#${domain}#${method}#${uri}#${agent}"
}

when HTTP_RESPONSE {
   set member_ip [IP::server_addr]
   set member_port [TCP::remote_port]
   set status [HTTP::status]
   set restimestamp [clock clicks -milliseconds]
   set responsetime [expr $restimestamp - $reqtimestamp]
   set content_length [HTTP::header "Content-Length"]
   if {$content_length equals ""} {
       set content_length "0"
   }
   set logString "$logString#${member_ip}#${member_port}#${status}#${responsetime}#${content_length}#"
   #log local0.info $logString
   set hsl [HSL::open -proto UDP -pool elklog]
   HSL::send $hsl "$logString"
}
```
Accessing Kibana interface
- Create Index patterns (find the one start with bigiphttplog-*)
- Go to "Discover" to see the raw/parsed log

## Customization
If you modify the irule, collect more information/field, you may need to modify the logstash config.
Below is the config file (10-syslog.conf) is bundled into the ELK image
```
input {
  tcp {
    type => "syslog"
    port => 5140
  }
  udp {
    type => "syslog"
    port => 5140
  }
}

filter {
  grok {
    match => { "message" => "<190>#%{NUMBER:timestamp}#%{DATA:virtual_name}#%{IP:client_ip}#%{POSINT:client_port}#%{IP:virtual_ip}#%{POSINT:virtual_port}#%{DATA:domain}#%{DATA:method}#%{DATA:uri}#%{DATA:agent}#%{IP:member_ip}#%{DATA:member_port}#%{POSINT:status}#%{DATA:responsetime}#%{DATA:contentlength}#"}
  }
  mutate {
    convert => { "contentlength" => "integer" }
  }
  mutate {
    convert => { "responsetime" => "integer" }
  }
    date {
        match => [ "timestamp" , "UNIX_MS"]
        target => "@timestamp"
    }

  geoip {
    source => "client_ip"
    database => "/etc/logstash/GeoLite2-City.mmdb"
  }
}
```
The most important part is the line start with ```match => { "message" =>```. You may want to use a Grok debugger like this one to develop your own: https://grokdebug.herokuapp.com/

Have fun!
