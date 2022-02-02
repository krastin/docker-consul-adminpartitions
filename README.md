# docker-consul-adminpartitions
A repository to experiment with Consul Admin partitions

# Requirements

- [Docker](https://docs.docker.com/get-docker/)
- A license for Consul

# How to set up

Create a `.env` file containing an export of the Consul License. The file should look like this, with the license replacing the text _LICENSE_HERE_ :

```
CONSUL_LICENSE=LICENSE_HERE
```

# How to run

Execute `docker-compose up --detach`.

Create a partition `partition1` so that `consul-client2`, which is a part of this partition, can join the cluster.

```
curl --request PUT \
   --header "X-Consul-Token: secret" \
   --data '{ "Name": "partition1" }' \
   http://localhost:8500/v1/partition
```

# Topology

The cluster comprises of three servers and three clients.

The bootstrap token for this cluster is `secret`.

One client is in a default partition, another client is in a separate partition, and the third client does not support partitions at all.

- consul-server1, running 1.11.1-ent, exposing ports 8500 and 8600 on localhost:8500 and localhost:8600
- consul-server2, running 1.11.1-ent
- consul-server3, running 1.11.1-ent
- consul-client1, running 1.11.1-ent, exposing port 8500 on localhost:8501
- consul-client2, running 1.11.1-ent, exposing port 8500 on localhost:8502, in partition `partition1`
- consul-client3, running 1.8.19-ent, exposing port 8500 on localhost:8503, not supporting partitions

# Testing endpoints

## Test TXN endpoint

- Create a permissive policy

```
curl --silent --request PUT --data @data/create-partition1-policy1.json http://127.0.0.1:8502/v1/acl/policy -H "X-Consul-Token: secret" | jq
{
  "ID": "c4a7aab2-fb61-e3ca-1a81-e83eb96166bd",
  "Name": "partition1-policy1",
  "Description": "",
  "Rules": "acl = \"write\"\r\n\r\npartition \"partition1\" {\r\n\r\n  acl = \"write\"\r\n  policy = \"write\"\r\n  mesh = \"write\"\r\n\r\n  key_prefix \"\" {\r\n    policy = \"write\"\r\n  }\r\n\r\n  node_prefix \"\" {\r\n    policy = \"write\"\r\n  }\r\n\r\n  session_prefix \"\" {\r\n    policy = \"write\"\r\n  }\r\n\r\n  service_prefix \"\" {\r\n    policy = \"write\"\r\n    intentions = \"write\"\r\n  }\r\n\r\n  namespace_prefix \"\" {\r\n    acl = \"write\"\r\n    policy = \"write\"\r\n\r\n    key_prefix \"\" {\r\n      policy = \"write\"\r\n    }\r\n\r\n    session_prefix \"\" {\r\n      policy = \"write\"\r\n    }\r\n\r\n    service_prefix \"\" {\r\n      policy = \"write\"\r\n    }\r\n\r\n    node_prefix \"\" {\r\n      policy = \"read\"\r\n    }\r\n  }\r\n}",
  "Hash": "tUrH6d5FzMJbd0LkKssKXDQRG/EWmnj6jbGTy7oj4Qk=",
  "Partition": "partition1",
  "Namespace": "default",
  "CreateIndex": 2066,
  "ModifyIndex": 2066
}
```

- Associate a token to the policy

```
export token1accessor=$(curl --silent --request PUT --data @data/create-partition1-token1.json http://127.0.0.1:8500/v1/acl/token -H "X-Consul-Token: secret" | jq -r .AccessorID)

export token1=$(curl --silent --request GET "http://127.0.0.1:8500/v1/acl/token/$token1accessor?partition=partition1" -H "X-Consul-Token: secret" | jq -r .SecretID)
```

- Attempt to inherit partition from token

```
$ # TXN does not infer partition from the token
$ curl -k -X PUT http://localhost:8502/v1/txn --data @data/txn-kv-set.json -H "X-Consul-Token: $token1"
{"Results":null,"Errors":[{"OpIndex":0,"What":"Permission denied"}]}% 

$ # KV does infer partition from the token
$ curl --request PUT --data 'test2' http://127.0.0.1:8502/v1/kv/test2  -H "X-Consul-Token: $token1"
true%
```

- Verify that token is a part of partition1

```
$ curl --silent --request GET "http://127.0.0.1:8500/v1/acl/token/$token1accessor?partition=partition1" -H "X-Consul-Token: secret" | jq -r .Partition         
partition1
```

