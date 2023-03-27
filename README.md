# CL-DEMO

A quick demo to introduce the advantages of using cluster linking.


## Topology

- broker_src and zk_src are in a private network `net_src`.
- broker_dest and zk_dest are in a private network - `net_dest`.
- broker_src can talk to broker_dest via `net_cl`
```
broker_src -----> (net_src: 2181) zk_src
  \
   (net_cl: *)
    \
   broker_dest -----> (net_dest: 2181) zk_dest
```

Start the clusters.
```
docker-compose -f docker-compose-dest.yml -f docker-compose-src.yml up -d
docker-compose -f docker-compose-dest.yml -f docker-compose-src.yml ps
```

## Tutorial

- Prepare a topic in `broker_src`.
    ```
    docker exec -it broker_src kafka-topics \
        --bootstrap-server broker_src:9092 \
        --create \
        --topic aaa \
        --partitions 12 \
        --config retention.ms=99999
    ```

- Create a simple CL with all default configurtions in `broker_dest`.
    ```
    docker exec -it broker_dest kafka-cluster-links \
        --bootstrap-server broker_dest:9092 \
        --create \
        --link cl-demo \
        --config bootstrap.servers=broker_src:9092
    ```

-  Create mirror topic in `broker_dest`.
    ```
    docker exec -it broker_dest kafka-mirrors \
        --bootstrap-server broker_dest:9092 \
        --create \
        --mirror-topic aaa \
        --link cl-demo
    ```

- Check the CL status.
    ```
    docker exec -it broker_dest kafka-replica-status \
        --bootstrap-server broker_dest:9092 \
        --topics aaa \
        --include-linked 
    ```

- The CL is set.
    
    Test - Produce data from broker_src, consume from both side.
    ```
    docker exec -it broker_src kafka-console-producer \
        --bootstrap-server broker_src:9092 --topic aaa
    docker exec -it broker_src kafka-console-consumer \
        --bootstrap-server broker_src:9092 --topic aaa --from-beginning
    docker exec -it broker_src kafka-console-consumer \
        --bootstrap-server broker_dest:9092 --topic aaa --from-beginning
    ```

    Test - Produce data from broker_dest.
    ```
    docker exec -it broker_dest kafka-console-producer \
        --bootstrap-server broker_dest:9092 --topic aaa
    ```

    Test - Change topic configuration.
    ```
    docker exec -it broker_src kafka-configs \
        --bootstrap-server broker_src:9092 \
        --alter \
        --topic aaa \
        --add-config retention.ms=88888
    docker exec -it broker_src kafka-topics \
        --bootstrap-server broker_src:9092 \
        --describe --topic aaa
    docker exec -it broker_src kafka-topics \
        --bootstrap-server broker_dest:9092 \
        --describe --topic aaa
    ```

    Test - Consumer group offsets.
    ```
    # generate a dummy data file, numbers from 1 to 9999
    file="numbers.txt"; if [ -f "$file" ]; then rm "$file"; fi; for i in {0..9999}; do echo -e "$i" >> "$file"; done

    # produce into topic
    docker exec -i broker_src kafka-console-producer \
        --bootstrap-server broker_src:9092 \
        --topic aaa < numbers.txt
    
    # consume from topic with consumer group id
    docker exec -it broker_src kafka-console-consumer \
        --bootstrap-server broker_src:9092 \
        --topic aaa \
        --from-beginning \
        --group mygroup

    # current group status - broker_src
    docker exec -it broker_src kafka-consumer-groups \
        --bootstrap-server broker_src:9092 \
        --describe \
        --group mygroup
    
    # current group status - broker_dest
    docker exec -it broker_dest kafka-consumer-groups \
        --bootstrap-server broker_dest:9092 \
        --describe \
        --group mygroup

    # alter cluster linking to 
    docker exec -it broker_dest kafka-configs \
        --bootstrap-server broker_dest:9092 \
        --alter \
        --cluster-link cl-demo \
        --add-config-file /configs/cl.config
    ```

## Cleanup
- Delete mirror
    ```
    docker exec -it broker_dest kafka-topics \
        --bootstrap-server broker_dest:9092 \
        --delete \
        --topic aaa 
    ```
- Delete CL
    ```
    docker exec -it broker_dest kafka-cluster-links \
        --bootstrap-server broker_dest:9092 \
        --delete \
        --link cl-demo
    ```
- Docker-compose down
    ```
    docker-compose -f docker-compose-dest.yml -f docker-compose-src.yml down
    ```

## Ref
- https://docs.confluent.io/platform/current/multi-dc-deployments/cluster-linking/topic-data-sharing.html#tutorial-using-cluster-linking-for-topic-data-sharing