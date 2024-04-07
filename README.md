```mermaid
flowchart TD
    subgraph S3_Bucket
        S3_Object_CSV[CSV File] --> Filebeat_CSV[Filebeat CSV Input]
        S3_Object_JSON[JSON File] --> Filebeat_JSON[Filebeat JSON Input]
        S3_Object_XML[XML File] --> Filebeat_XML[Filebeat XML Input]
    end
    subgraph Filebeat
        Filebeat_CSV -- Logs --> Logstash_CSV[Logstash]
        Filebeat_JSON -- Logs --> Logstash_JSON[Logstash]
        Filebeat_XML -- Logs --> Logstash_XML[Logstash]
    end
    subgraph Logstash_CSV
        Logstash_CSV -- Filter --> CSV_Processor[CSV Processor]
        CSV_Processor -- Filter --> Grok_Filter_CSV[Grok Filter]
        Grok_Filter_CSV -- Filter --> Elasticsearch[Elasticsearch]
    end
    subgraph Logstash_JSON
        Logstash_JSON -- Filter --> JSON_Processor[JSON Processor]
        JSON_Processor -- Filter --> Grok_Filter_JSON[Grok Filter]
        Grok_Filter_JSON -- Filter --> Elasticsearch[Elasticsearch]
    end
    subgraph Logstash_XML
        Logstash_XML -- Filter --> XML_Processor[XML Processor]
        XML_Processor -- Filter --> Grok_Filter_XML[Grok Filter]
        Grok_Filter_XML -- Filter --> Elasticsearch[Elasticsearch]
    end
    subgraph Elasticsearch
        Elasticsearch -- Query --> Kibana[Kibana]
    end
    subgraph Kibana
        Kibana -- Visualization --> Dashboard[Dashboard]
    end
```





# Configuring the Elastic stack

When first spinning up elasticsearch, you'll want to run some setup scripts, which are included as part of a setup container in the docker-compose config. Simply uncomment it, run `docker-compose up -d`, then `docker compose down` uncomment it, and run `docker compose up -d` again.




## Filebeat

Filebeat scans the data sources and forwards info to logstash, which can then filter the incoming stream before sending it to elasticsearch

Configure its settings in filebeat.yml. Use the file in this link as a reference https://raw.githubusercontent.com/devopsschool-demo-labs-projects/elasticsearch/master/filebeat-config-file/filebeat.reference.yml 

Use this documentation to configure filebeat
https://www.elastic.co/guide/en/cloud/current/ec-getting-started-search-use-cases-beats-logstash.html#ec-beats-logstash-filebeat


To make a new field after data has been ingested, start with this block

```
def message = doc['aws.s3.object.key.keyword'].size() > 0 ? doc['aws.s3.object.key.keyword'].value : null;
if (message != null) {
    def match = /.*(\d{5}).*\..*/.matcher(message);
    if (match.find()) {
        emit(match.group(1));
        return;
    }
}
```

And replace match = {} with a regular expression capturing the field. The above extracts any number of 5 digits. To match the task label in a bids format, you could use

```
def message = doc['aws.s3.object.key.keyword'].size() > 0 ? doc['aws.s3.object.key.keyword'].value : null;
if (message != null) {
    def match = /.*task-(.*)_.*\..*/.matcher(message);
    if (match.find()) {
        emit(match.group(1));
        return;
    }
}
```

this will match any string that contains _task-(captured)_ and capture the part in parentheses for the field value.

In Kibana data views, select "Add a field" at the bottom of the page and place the above into the section "Set value"

# Using the Python Elasticsearch Client

```
from elasticsearch import Elasticsearch

# Password for the 'elastic' user generated by Elasticsearch
ELASTIC_PASSWORD = "<password>"

# Create the client instance
client = Elasticsearch(
    "https://localhost:9200",
    ca_certs="/path/to/http_ca.crt",
    basic_auth=("elastic", ELASTIC_PASSWORD)
)

# Successful response!
client.info()
# {'name': 'instance-0000000000', 'cluster_name': ...}
```

# Configuring Ingest Inputs in Filebeat

Filebeat handles the scanning of data on local and S3 storage. You can set rules on what it tries to read. In `filebeat/inputs/*.yml`, exclude filetypes that cannot be interpretted or expressed in text, such as MRI, audio, and video files. 

You can also set patterns on which file extensions you want to scan, and a prefix path to only set an input to scan a specific directory.

## Index files but drop or truncate their contents

If you want to include records of objects but don't care about their contents, you can drop fields that match a regex pattern with the `drop_fields` or `truncate_fields` processors in `filebeat.yml`

Files with multiple lines that aren't structured may end up creating hundreds or thousands of events for a single object, even if the fields are dropped, such as media files. To simply log the existence of files, the best way is to index a special bucket containing outputs to `find` command, and make an index just for that.

This way, you can search all of your buckets and directories with free-text without having to wait for the command to complete. Instead, schedule a task periodically in off-peak hours like `mc find path/to/s3 > bucket-name.txt && mc cp bucket-name.txt path/to/buckets-manifest`

As a filebeats input, scan files matching `*manifest.txt` from the manifests bucket then you can rapidly search for files of special type by name in free-text without expending resources trying to read their content.


## Aggregate documents accross indices into new indices
In the `transforms` directory you can create rules and mappings to filter and funnel your data into more structured indices