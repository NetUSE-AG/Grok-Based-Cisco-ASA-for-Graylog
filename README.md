# Grok-Based-Cisco-ASA-for-Graylog
This article will help you to use your Graylog to make sense of logs from Cisco ASA.
The focus here is to maximize the performance of you Graylog and get the most out of your logs. The extracted fields will adhere to [GIM](https://schema.graylog.org/en/stable/) as much as possible. 

# We help you - you help us
If you see things from this repo not working and maybe even found a fix for the problem - let us know so we can improve this. We have tested this collection on multiple instances, but still observed differences at some point. Please make sure to add some samples as well - but please without PII.
	

# Prefix: how we parse logs
When using Grok / regex to parse your logs, you should be careful not to waste a lot of your CPU. There is a wonderful post in the [blog by Ben Caller](https://blog.doyensec.com/2021/03/11/regexploit.html) about this topic. In short: Don't use greedy operators, which are forced to do backtracking. This will ruin your performance.

Our way to deal with this is a big collection of Grok-Pattern named ```NU_DATA_ALL_BUT_*```. Those are greedy - except for one letter, which is the delimiter we are looking for. Our Grok collection utilizes this type of patterns.

I need to do a little excourse to our standard-processing-schema. Our processing happens exclusively via pipelines, we do not use extractors or stream rules in Graylog.
The first pipeline ```[proc] normalization``` is attached to the ```Default Stream```. Here the parsing of _all_ messages happens. The last rule takes all messages and routes them into the next stream - the ```[proc] normalized```. There is another pipeline called ```[proc] enrichment```, attached to this stream. This pipeline adds external information, such as reverse dns, geo-info, and so on and routes all messages in the last stage to the stream ```[proc] enriched```. On this last stream we again have a pipeline called ```[proc]routing``` attached. Here we distribute the logs into different final streams.

Here is a visualisation of this model:
![Graylog Processing Model](Images/Processing-Model.png)

#Fieldnames and Types
To make the most of your logs you will need to change the types of certain fields. IP-fields will be stored as IP - and therefore be searchable via CIDR-notation. There are a few fields of type integer as well - bytes transferred and so on. Those will be stored as long/unsigned long and because of that we are able to calculate sums on them.

## find out the index-set
In the folder "FieldMappings" you will find a json-file called ```asa-custom-mapping.json```. In line four, you will see the name/prefix of the index this mapping applies to. To find you prefix go to the overview of streams. Here, you will see the index-set your stream for Cisco ASA lives on. Now go to System/Indices and search for that index set. You will be able to see the ```Index prefix``` here. Replace the current value in the ```asa-custom-mapping.json``` with that value.

## apply the index-set
To apply the asa-custom-mapping.json we will need to do a curl-request to your Opensearch. 

### find your Opensearch host, user and pwd
To interact with Opensearch, credentials are required. These can be found in your server.conf on the line starting with ```elasticsearch_hosts```
```
/etc/graylog/server# cat server.conf | grep elasticsearch_hosts
```

### change the config in Opensearch
1) Assume, the file ```graylog-custom-mapping.json``` is located in the current folder of your terminal.
2) Customize the following command with: 
* the right username
* the right password
* the right IP/domain of your openseach host
from the step above. Then change the config in Opensearch:
```
curl -X PUT -d @'graylog-custom-mapping.json' -H "Content-Type: application/json" --insecure 'https://username:password@openseach.host:9200/_template/graylog-custom-mapping?pretty'
```
Congratulations, you've changed the types of the fields included in the json-file! You might want to run this from your Graylog-node, as access to Opensearch should not be available from elsewhere. 

### Maintenance
Now you changed your field mapping - you might run into errors in your indexing. If your Graylog tries to write ```ssh``` in the field destination_port, it will try to interpret it as an integer - and fail. You can see this on the ```System/Overview``` page in your Graylog. Hopefully your Overview will always look like this:

![Indexing](Images/Indexing_Error.png)

If not, you will see errors like these:
```
ElasticsearchException[Elasticsearch exception [type=mapper_parsing_exception, reason=failed to parse field [fsize] of type [long] in document with id 's0me-rand0m-iD-from-Graylog'. Preview of field's value: '18302908161605107712']]; nested: ElasticsearchException[Elasticsearch exception [type=input_coercion_exception, reason=Numeric value (18302908161605107712) out of range of long (-9223372036854775808 - 9223372036854775807) at [Source: (byte[])"{"msg":"[...]""[truncated 1512 bytes]; line: 1, column: 1742]]];

OpenSearchException[OpenSearch exception [type=mapper_parsing_exception, reason=failed to parse field [winlog_event_data_param1] of type [date] in document with id 's0me-rand0m-iD-from-Graylog'. Preview of field's value: 'Windows Update']]; nested: OpenSearchException[OpenSearch exception [type=illegal_argument_exception, reason=failed to parse date field [Windows Update] with format [strict_date_optional_time||epoch_millis]]]; nested: OpenSearchException[OpenSearch exception [type=date_time_parse_exception, reason=Failed to parse with all enclosed parsers]];
```
If you run into such errors, find the field with a value not fitting to the type of the field and build a rule similar to "ASA ssh-renaming"

```
rule "ASA ssh-renaming"
when
  has_field("destination_port") &&
  to_string($message.destination_port) == "ssh"
then
  set_field(field:"destination_port", value:22);
end
```
This will change the value of the field "destination_port" to "22" if it's "ssh" in the original log.


# Content packs for parsing Cisco ASA
## Installing the content pack
In the folder ```Content Packs``` you will find a content pack as a json-file. This content pack includes all necessary Grok-patterns as well as the necessary rules to implement them.
To install the content pack to on ```System/Content Packs``` and click on ```Upload``` in the upper right. Choose the file upload the content pack.

## Adding the Rules to the Pipeline
To stay in the schema from above open the pipeline ```[proc] Normalization```. The processing will happen in those stages:
1) Add here the rule named ```ASA_BASE``` in stage x. This rule shortens the logs by their header to make parsing more consistent across different setups and gets us the field ```vendor_syslog_id```, which is used as a condition for the rules in stage x+2. Here you will need to do an adjustment: add the ID of your Input, where Cisco ASA is ingested. This is important, otherwise the logs will not find their way into the parsing.
2) in stage x+1 add the rule ```ASA_Prefix```. This will parse the prefix of your messages. Depending on the configuration of your Cisco ASA (rerouted via syslog-server, syslog config changes, ...) you will need to adjust things there to correctly parse the hostname from the syslog header correctly. 
3) Add all the rules named like ```ASA_VPN_sixDidgets_description``` into stage x+2. This will be quite a lot of work, as there are approximately 160 rules. Copy ```ASA_VPN``` into your clipboards and paste it every time searching. This will do the parsing for the different message-types.
4) add the rules ```ASA https-renaming```and ```ASA ssh-renaming``` into stage x+3. Those will fix some inconsistencies in logging by Cisco ASA.

## monitor unpared logs
Create yourself a Dashboard to monitor unparsed logs from Cisco ASA. We assume the filed ```vendor_syslog_id``` is set based on the base-pattern. Then search for messages not containing any parsed fields like this:

```
_exists_:vendor_syslog_id AND NOT _exists_:source_ip AND NOT _exists_:user_name
```

You might need to add a few more fieldnames if the parsing is working as intended.


# Use cases / Dashboards
This is a list of Use Cases for Logs from Cisco ASA, it is neither complete, nor finished. If you have more ideas, please contribute:
* with regards to Anyconnect VPN
  * look for ```vendor_syslog_id:113019``` - the logout summary. You will be able to see how long the users were logged in for and how much up/download they've done in that session.
  * collect ```vendor_syslog_id:734003``` and process it: create a rule setting a field with the name ```session_attribute_key``` and the value ```session_attribute_value```. Collect those logs into single logs using our [Context Collector](https://github.com/NetUSE-AG/graylog-plugin-context-collector) 
  -- Add geo-coordinates to ```source_ip``` and ```destination_ip```. Then search for ```_exists_:source_geo_coordinates AND _exists_:vpn_login_status``` and create a geo-map of logins.
  * look for vendor_syslog_id:722051 to see the external IP of your user and the internal one in the same log
  * ```vendor_syslog_id:725012``` contains a field ```certificate_cipher```. This contains the used cipher.
* general firewalling. You will be able to search for outdated AnyConnect Clients 
  * Look for ```vendor_syslog_id:302013 (TCP) 302015 (UDP) and AND destination_port:3389``` to see allowed RCP Connections. A heatmap is awesome to do this!
  * look for ID 106015/106102 for denied connections. A heatmap is wonderful for this.



