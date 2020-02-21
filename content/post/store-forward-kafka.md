+++
date  = "2018-09-08T19:09:58+01:00"
draft = false
title = "Store and forward with kafka"
tags  = [ "kafka", "pro-tip"]
+++

Store and forward is a technique normally applied in hardware-routing and networking to avoid package-loss. If we apply the same technique when sending records to kafka, we dont have to deal downtime of your kafka-cluster. 

## Store and forward

"Store and forward" is a technique used widely in telecommunications and in router technology. The idea is simple: when we want to send a packet, instead of sending it directly, the packet is stored in a cache or a local storage and sent to the network at a later point. If the transmission of the package is successful, the package is deleted from the local storage. If the transmission failed in some way, the package remains in the storage and is retried at a later point. 

## Applied to kafka producers

This same technique can be applied to any kafka-producer when appending records to kafka. Instead of adding the record directly to kafka, the records are stored as serialized objects in a temporary database along with the topic they should be appended to. A scheduled task should check if there are any records in the temporary database and deserialize before sending them off on the destined topic. If the send is succesful, the record is deleted from the temporary database. If the send is unsuccesful, the records remain untouched and will be picked up by the scheduled task at a later point.

## A note on avro (or any other schema)

If you are using avro as the wire-format for your records, you probably want to store the schema-name and version along with the serialized version of the record for debugging purposes. 

## Benefits

When using this technique/pattern you can sit back and relax when your kafka-cluster is down due to maintenance or having troubles for one reason or the other. The producing applications does not have to be taken down and will recover the minute the kafka-cluster is back up. 