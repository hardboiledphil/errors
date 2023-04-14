IncompatibleClassChangeError
JBOSS EAP 7
WILDFLY
JAVA
KAFKA
SERIALIZATION


I had an enterprise application running in JBoss 7 EAP with multiple modules including multiple war files for REST endpoints.

root
 --> adminservice (containing objectmodel to be marshalled)
 --> json-webapp (containing the endpoint to do the send to Kafka)
 --> service-ejb
 --> service-ear
 --> etc
 
The goal was to introduce into some of these endpoints in the webapp the ability to be able to marshall the requests to a Kafka instance for later replay.

I was getting IncompatibleClassChangeError which was strange as I wasn't changing anything on the class as it was passed into the KafkaProducer.  Using the MockProducer or even using the real producer in a unit test scenario was working - the object was being serialised by my custom json serializer.

In the end I created a generic AcmeObject with a couple of properties and low and behold it worked.  Ok now to find out why my real object didn't work.  I soon realised that the object that was working was in the web-app and so was the custom serializer.  The failing objects/serializers were in the adminservice.  After moving the AcmeObject and serializer to the admin service I could see they were failing as well.  Hmm. So I moved the serializer classes to the json-webapp module and finally they worked.

'''
  ProducerRecord<String, RequestReplay> record
          = new ProducerRecord<>(CHAIN_RULE_TOPIC, chainRuleRequest.getKey(), chainRuleRequest);
  kafkaProducer.send(record, (recordMetadata, e) -> {
      if (e == null) {
        // successful completion
      } else {
        // send failed - error in e
      }
  });'''
'''
package com.stuff.json;

import org.apache.kafka.common.errors.SerializationException;
import org.apache.kafka.common.serialization.Serializer;
import org.codehaus.jackson.map.ObjectMapper;

public class RequestReplaySerializer implements Serializer<RequestReplay> {

    private final static ObjectMapper om = new ObjectMapper();

    public RequestReplaySerializer() {}

    @Override
    public byte[] serialize(String topic, RequestReplay data) {
        if (data == null) {
            return null;
        } else {
            try {
                return om.writeValueAsString(data).getBytes();
            } catch (Exception e) {
                throw new SerializationException("Error serialising object", e);
            }
        }
    }
}'''

In pom.xml

'''  <dependency>
      <groupId>org.apache.kafka</groupId>
      <artifactId>kafka-clients</artifactId>
      <version>2.5.1</version>
  </dependency>'''

And also for testing this client library gives a mockproducer

'''   MockProducer<String, RequestReplay> mockProducer
           = new MockProducer<>(true, new StringSerializer(), new RequestReplaySerializer());
                
   assertEquals(1, mockProducer.history().size());'''
