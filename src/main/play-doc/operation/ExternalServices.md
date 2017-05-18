# Service Discovery for External Services

ConductR's service locator can be configured to resolve service names against external services. This 
becomes particularly useful when needing to connect with existing instances of various persistence technologies 
including Kafka, Cassandra and Elasticsearch. ConductR provides a configuration structure that declares the 
translations between a service name and its external addresses. External
addresses are provided as an array and can be one of a resolved address,
a unresolved DNS name along with a port or an unresolved [DNS SRV](https://en.wikipedia.org/wiki/SRV_record) name.
A URI scheme for demarking DNS SRV records is provided and takes inspiration from
https://tools.ietf.org/html/draft-jennings-http-srv-03. In addition to
`http+srv` and `https+srv`, `tcp+srv` and `udp+srv` are also permitted. Note that
DNS SRV names are often not valid hostnames given the use of the underscore
character. However we interpret DNS SRV hosts using a URI's authority,
additionally adopting the URI convention of userinfo within authority.
Note that user info and paths may be specified as per URI conventions.
Examples for all 3 types:

    external-service-addresses {
      "elastic-search" = [
        "http://10.0.1.1:9005",                                                 resolved address
        "https://user@pass:my-elastic-search.com:9000/some-path-in",            unresolved DNS
        "https+srv://_client-port._elasticsearch-executor._tcp.elasticsearch.mesos" # unresolved DNS SRV
      ]
    }

For a concrete example, let's suppose that you have an Elasticsearch service hosted at https://myelasticsearch.com with a username of `myuserid` and a password of `mypassword`. You can configure ConductR's `conf/conductr.ini` file with the following in order to reach this external service:

    -Dconductr.service-locator-server.external-service-addresses.elastic-search.0=http://myuserid:mypassword@https://myelasticsearch.com:9200
    
