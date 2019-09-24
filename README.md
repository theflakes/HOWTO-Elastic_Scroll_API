# HOWTO-Elastic_Scroll_API
Example Python how to use the Elastic Scroll API to pull back more than the default max of 10k logs.  

This is example code, it is not meant to work as a simple copy paste.

```
import getpass
import urllib3
from elasticsearch import Elasticsearch

# disable warning if you are using self-signed certs and stuff
urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)


def do_something_with_logs(logs):
    # process set of logs here


# setup Elastic connection params
es = Elasticsearch(
        ['https://ELK:9200'],
        http_auth=(input("Username: "), getpass.getpass("Passsword: ")),
        scheme="https",
        use_ssl=True,
        verify_certs=True,
        ssl_assert_hostname=False,
        ca_certs='cert.pem',
        http_compress=True,
        port=9200
    )

# initial ELK search
# we'll return 1000 results with first search and all subsequent scroll searches
search_string = ("""
    {
        "size": 1000,
        "query": {
            "bool": {
                "must": [
                    {"term": { "field": "Search for this string." }}
                ]
            }
        }
    }""")
      
# Two ways of running the initial query, we'll use POST
# note the initial query uses the param "?scroll=1m" which tells Elastic to hold results for 1 minute
# logs = self.es.search(index="logstash*", body=search_string, scroll='1m', size=1000, request_timeout=30)
logs = es.transport.perform_request(method='POST', url='/logstash*/_search?scroll=1m', body=search_string)

# how many logs were returned?
scroll_size = len(logs['hits']['hits'])

# Loop through the ELK queries' scroll set till there are no more logs
while scroll_size > 0:
    # process the set of logs
    do_something_with_logs(logs)
    
    # get the scroll_id so we can get the next batch of logs, if there are any
    sid = logs['_scroll_id']
    
    # build the POST body to retrieve the next set of logs using the above scroll_id
    body = ("""
        {
          "scroll": "1m",
          "scroll_id": "%s"
        }
        """ % (sid))
      
    # you should use POST as scroll_id can be huge, a huge scroll_id will, over 4k, will error out
    # note that to retreive the rest of the results after the initial set, use the "/_search/scroll" uri
    logs = self.es.transport.perform_request(method='POST', url='/_search/scroll', body=body)
    
    # how many logs were returned?, if 0 then while loop will exit
    scroll_size = len(logs['hits']['hits'])
```
