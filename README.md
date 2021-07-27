# elk-bigip-http-log
Sending log from a bigip http/https virtual server to a dockerized ELK via HSL
## Starting the ELK container
Before we start, add the following line into /etc/sysctl.conf
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
Accessing Kibana interface
- Create Index patterns (find the one start with bigiphttplog-*)
- Go to "Discover" to see the raw/parsed log
