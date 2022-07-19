# Rabbitmq Cluster

## Recovering  configuration for a Split-Brain

- this way,if face problem need to restart

```conf
cluster_partition_handling = pause_if_all_down

## Recovery strategy. Can be either 'autoheal' or 'ignore'
cluster_partition_handling.pause_if_all_down.recover = autoheal

## Node names to check
cluster_partition_handling.pause_if_all_down.nodes.1 = rabbit@rabbitmq1
cluster_partition_handling.pause_if_all_down.nodes.2 = rabbit@rabbitmq2
```

- this way,if face problem do not need to restart

```conf
cluster_partition_handling = autoheal

## Recovery strategy. Can be either 'autoheal' or 'ignore'
cluster_partition_handling.pause_if_all_down.recover = autoheal

## Node names to check
cluster_partition_handling.pause_if_all_down.nodes.1 = rabbit@rabbitmq1
cluster_partition_handling.pause_if_all_down.nodes.2 = rabbit@rabbitmq2
```

- but the two above is not recommended,they are for unstable network
in fact we do not need them ,when we deply all at on engine room
if we do not config for it,default is igonre

- docker ps only can check program if is starting,rabbitmq based on erlang,when rabbitmq is down,erlang is up,if you want to check status,you should use admin web
or command at each node

```shell
# maybe name need to change
docker exec -it rabbitmq bash
rabbitmqctl cluster status 
```

- if you do not want to solve problem in manual,choose first one

- cluster_partition_handling config for us is choose one node to as last node to reject to accept the requests

## Recovering manually for a Split-Brain

1. we need check which node is last one to accept the requests
if we had recovering configuration pause_if_all_down

2. choose the node which one could start and accept the last request from client
you can use rabbitmq admin web to check,click the exchanges,
check each exchange,if they are empty

3. if you are not very sure,just choose one node can start to keep data,then backup the other 2 at Overview/Export definitions

4. restart the other 2 nodes,if they can recovery by itself,all step is done

```shell
docker restart rabbitmq
```

4. if can not recovery by itself,just delete this nodes and recreate for it

```shell
docker rm -f rabbitmq
rm -rf /data/rabbitmq/data
```

5. then follow the deployment guide to create it again

6. Overview/Import definitions to import old data
