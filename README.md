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