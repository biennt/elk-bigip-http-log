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
- Create a pool named elkpool, pointing to the machine running ELK above, destination port is 5140
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
