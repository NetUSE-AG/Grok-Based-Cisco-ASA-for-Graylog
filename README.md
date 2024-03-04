# Grok-Based-Cisco-ASA-for-Graylog
This article will help you to use your Graylog to make sense of logs from Cisco ASA.
The focus here is to maximize the performance of you Graylog and get the most out of you logs. The extracted field will stick to the [GIM](https://schema.graylog.org/en/stable/) as good as possible. 

# We help you - you help us
If you see things from this repo not working and found a fix for the problem - let us know so we can improve this. We have tested this collection with multiple instances, but still observed differences at some point.
	

#Prefix: how we parse logs
If you use Groks / regex to parse your logs you should be carefull not to waste a lot of your CPU. There is a wonderfull post in the [blog by Ben Caller](https://blog.doyensec.com/2021/03/11/regexploit.html) about this topic. In short: don't use greedy operators, which are forced to do backtracking. This will ruin your performance.

Our way to deal with this is a big collection of Grok-Pattern named ```NU_DATA_ALL_BUT_*```. Those are greedy - exept for one letter, which is the delmimiter we are looking for. Our Grok-collection utilizes those kind of patterns.

I need to do a little excourse to our standard-processing-schema. Our processing happens only via pipelines, we do not use extractors or stream rules.
The first pipeline ```[proc] normalization``` is attached to the ```Default Stream```. Here the parsing of _all_ messages happens. The last rule takes all messages and routes them into the next stream - the ```[proc] normalized```. To this stream there is another pipeline attached - called ```[proc] enritchment```. This pipeline add external information as reverse dns, geo-info, and so on and routes all messages in the last stage to the stream ```[proc] enriched```. On this last stream we again have a pipeline attached, calles ```[proc]routing```. Here we split up the logs into different final streams.

Here is a visualisation of this model:
![Graylog Processing Model](Images/Processing-Model.png)

#Fieldnames and Types
To make the most of you logs you will nee to change the type of some of the field to certain types. IP-fields will be stored as IP - and therefore be seachable with CIDR. There are a few field with integers as well - as bytes transfered and so on. Those will be stored ad long/unsigned long and therefore we are able to calculate sums on them.

## find out the index-set
In the folder "FieldMappings" you will find a jason-file called ```asa-custom-mapping.json```. In line four will see the name of the index this mapping applies to. To find you prefix go to the overview of streams. Here you will see the index-set your Stream for Cisco ASA lives on. Now go to /system/indices/ and search for that index set. You will be able to see the ```Index prefix``` here. Replace the current value in the ```asa-custom-mapping.json``` with that value.

## apply the index set
To apply the asa-custom-mapping.json we will need to do a curl-request to your Opensearch. 
### find your Opensearch host, user and pwd
To talk to your openseach you will need the access to your Opensarch. This can be found in your server.conf in the line with ```elasticsearch_hosts```
```
/etc/graylog/server# cat server.conf | grep elasticsearch_hosts
```

### change the config in Opensearch
1) Assume, the file ```graylog-custom-mapping.json``` is in the same folder as your terminal.
2) customize the following command with: 
* the right username
* the right password
* the right host
from the step above. Then change the config in Opensearch:
```
curl -X PUT -d @'graylog-custom-mapping.json' -H "Content-Type: application/json" --insecure 'https://username:password@openseach.host:9200/_template/graylog-custom-mapping?pretty'
```
Congratulations, you changed the fieldmapping of those field from the json! You might run this from your Graylog-Node, as access to Opensearch should not be available from the outside. 

### Maintainance
Now you changed your filed mapping - you might run into errors in your indexing. If your Graylog tries to write ```ssh``` in the field destination_port it will run into an integer - and fail. You can see this on the ```/system/overview``` page in your Graylog. Hopefully your Overview will always look like this:

![Indexing](Images/Indexing_Error.png)

If not, you will we an error like this:
```
ElasticsearchException[Elasticsearch exception [type=mapper_parsing_exception, reason=failed to parse field [fsize] of type [long] in document with id 's0me-rand0m-iD-from-Graylog'. Preview of field's value: '18302908161605107712']]; nested: ElasticsearchException[Elasticsearch exception [type=input_coercion_exception, reason=Numeric value (18302908161605107712) out of range of long (-9223372036854775808 - 9223372036854775807) at [Source: (byte[])"{"msg":"[...]""[truncated 1512 bytes]; line: 1, column: 1742]]];

OpenSearchException[OpenSearch exception [type=mapper_parsing_exception, reason=failed to parse field [winlog_event_data_param1] of type [date] in document with id 's0me-rand0m-iD-from-Graylog'. Preview of field's value: 'Windows Update']]; nested: OpenSearchException[OpenSearch exception [type=illegal_argument_exception, reason=failed to parse date field [Windows Update] with format [strict_date_optional_time||epoch_millis]]]; nested: OpenSearchException[OpenSearch exception [type=date_time_parse_exception, reason=Failed to parse with all enclosed parsers]];
```

# Content packs for parsing Cisco ASA
## Installing the Contentpack
In the folder ```Content Packs``` you will find a file with a contentpack. This contentpack includes all neccesary Grok-patterns as well as the neccesary rules to implement them.
To install the content pack to on ```/system/contentpacks``` and click on ```Upload``` in the upper right. Choose the file upload the content pack.

## Adding the Rules to the Pipeline
To stay in the schema from above open the pipeline ```[proc] Normalization```. The processing will happen in those stages:
1) Add here the rule named ```ASA_BASE``` in stage x. Here you will need to do an adjustment: add the ID of your Input, where Cicso ASA is ingested. This is important, otherwise the logs will not find their way into the parsing.
2) in stage x+1 add the rule ```ASA_Prefix```. This will adust the prefix of the messages. Depending on your configuration of your Cisco ASA (rerouted via syslog-server, syslog config changes, ...) you will need to adjust things there. 
3) Add all the rules named like ```ASA_VPN_sixDidgets_description``` into stage x+2. This will be quite a lot of work , as there are approx 160 rules. Copy ```ASA_VPN``` into your clipboards and paste it every time searching
4) add the rules ```ASA https-renaming```and ```ASA ssh-renaming``` into stage x+3. Those will fix some inconsistent logging by Cisco ASA.

## monitor unpared logs
Create yourself a Dashboard to monitor unparsed logs from Cisco ASA. We assume the filed ```vendor_syslos_id``` is set based on the base-pattern. Then search for messages not containing any fields with parsed filed like this:

```
_exists_:vendor_syslog_id AND NOT _exists_:source_ip AND NOT _exists_:user_name
```

You might need to add a few more fieldnames, if the parsing is working as it should.


# Use Cases / Dashbaords
This is a list of Use Cases for Logs from Cisco ASA, it it not complete, nor finished. If you have more ideas please contribute:
* with regards to Anyconnect VPN
  -- look for ```vendor_syslog_id:113019``` - the logout summary. You will be able to spot, how long the users
  -- collect ```vendor_syslog_id:734003``` and process it: create a rule setting a filed with the name of ```session_attribute_key``` and the value of ```session_attribute_value```. Collect those logs into single logs using our [Context Collector](https://github.com/NetUSE-AG/graylog-plugin-context-collector) 
  -- Add geo-coordinates to ```source_ip``` and ```destination_ip```. Then search for ```_exists_:source_geo_coordinates AND _exists_:vpn_login_status``` and create a geo-map of logins.
* general firewalling. You will be able to search for outdated AnyConnect Clients 
  ** Look for ```vendor_syslog_id:302013 AND destination_port:3389``` to see allowed RCP Connections. A heatmap is awesome to do this!











