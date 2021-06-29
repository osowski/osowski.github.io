---
title: 'Apache Kafka security - the Rosetta Stone to the event-streaming landscape!'
date: 2021-04-27T12:30:00-05:00
author: Rick Osowski
layout: post
permalink: /2021/04/kafka-security-fundamentals-1/
canonical_url: 'https://rosowski.medium.com/kafka-security-fundamentals-the-rosetta-stone-to-your-event-streaming-infrastructure-518f49640db4'
categories:
  - Technology
tags:
  - Apache Kafka
  - Event-Driven Architecture
  - Event Streaming
---


Understanding [Apache Kafka](https://kafka.apache.org/) security is much like discovering the Rosetta Stone to the entire event-streaming landscape. This is not meant to be hyperbole at all, since once you understand a few common core configurations, you have the ability to connect any Kafka-based, event-driven application to dozens of Kafka offerings with relatively few changes. The vast majority of these changes come in the form of Kafka broker endpoints, as well as authentication configuration information in the form of protocol selection and providing credentials in some fashion. This blog post will cover the six most important Kafka security configuration options from a Kafka client perspective _(based on [my team's experiences](https://ibm-cloud-architecture.github.io/refarch-eda/) so far)_ to allow you to become as flexible as a Swiss Army Knife when working with any Kafka offering.

As this is a deeper dive into Apache Kafka and the capabilities it provides, I will not be covering any introductory Kafka material here. You can head over to [kafka.apache.org](https://kafka.apache.org) for all the foundational information necessary or check out my friend Whitney's [What is Kafka?](https://www.youtube.com/watch?v=aj9CDZm0Glc&t=1s) video to learn everything you need to get started in under 10 minutes. Once you've read through everything here, if you still have questions or want to dig in further, the [Security](http://kafka.apache.org/documentation/#security) section of the official Kafka documentation covers everything in depth with a great deal of clarity - so that would be my first recommendation if you read through here to find that none of the common configurations below match your expected environment setups!

Before we jump in, there are two caveats to address on the scope of this blog series:

- First, there are many, many ways to configure Kafka security from a Kafka administrator point-of-view. The focus of this short blog series is to more effectively comprehend Kafka security as a Kafka client (producer or consumer) and not a Kafka administrator (operations / provider). For a deeper understanding of practical Kafka operational security from an adminstrator's point of view (beyond the aforementioned [Security](http://kafka.apache.org/documentation/#security) docs above), you can reference Confluent's [Security Overview](https://docs.confluent.io/platform/current/security/general-overview.html) for a much deeper understanding on how on to securely deploy and maintain Kafka well into the future!

- Second, there are two different patterns for authenticating with Kafka. The one we will address in this first post focuses on username and password-based authentication mechanisms. The other, which we will cover in a follow-up post, focuses on using SSL certificates for authentication, above and beyond their use in traditional encryption mechanisms. This latter pattern is traditionally referred to as mutual TLS-based authentication or mTLS, for short. If your current Kafka scenario doesn't provide you with user-based credentials, don't worry, we'll cover that soon. The majority of this blog post will still hold true, with a few changes to the core six configuration properties covered below.

Without any further ado, let's get started!

## Kafka Settings

- [**bootstrap.servers**](http://kafka.apache.org/documentation/#adminclientconfigs_bootstrap.servers)

  This is the most important setting as it tells Kafka clients where to look to talk to your applications. Sometimes this will be a string that contains just a single `hostname:port` combination, while other times it will be a comma-separated list of `hostname:port` combinations. Depending upon the Kafka offering you are connecting to, as well as how and where it is deployed, you will either be providing a singular `bootstrap` address or the entire collection of Kafka brokers in the same string. The `bootstrap` endpoint is a functional intermediary that Kafka offerings provide to allow for simpler networking in cases of expected fluctuations in the number of Kafka brokers, as it is a single URL that will return all the active brokers in the cluster to your clients automatically.

- [**security.protocol**](http://kafka.apache.org/documentation/#adminclientconfigs_security.protocol)

  This is the next most important security setting as it tells the clients how to communicate with the brokers. If you've ever had HTTPS issues while using a web browser, this is the setting that would give Kafka clients SSL issues while attempting to connect to Kafka brokers! If you are receiving errors in your application while attempting to connect to Kafka with references to a `broker (-1)`, you will usually need to revist this setting _(i.e. you should be using `SASL_SSL` but you configured `PLAINTEXT`, etc.)_.

  The valid values are:
  - `PLAINTEXT` _(using PLAINTEXT transport layer & no authentication - default value)_
  - `SSL` _(using SSL transport layer & certificate-based authentication)_
  - `SASL_PLAINTEXT` _(using PLAINTEXT transport layer & SASL-based authentication)_
  - `SASL_SSL` _(using SSL transport layer & SASL-based authentication)_

- [**ssl.truststore.location**](http://kafka.apache.org/documentation/#adminclientconfigs_ssl.truststore.location)

  To allow for encryption of communication between Kafka brokers and clients, as specified by our `security.protocol` setting above, we need to provide our Kafka clients with the location of a trusted Certificate Authority-based certificate. This file is often provided by the Kafka administrator and is generally unique to the specific Kafka cluster deployment. It is outside the scope of this blog post to determine how you provide access to this file for your individual runtimes, but it must exist inside the runtime container somewhere.

  For Java-based clients, this file needs to be in the JKS format. For Python or NodeJS-based clients, this file should be in either a PEM or P12 format. To extract a PEM-based certificate from a JKS-based truststore, you can use the following command:
  `keytool -exportcert -keypass {truststore-password} -keystore {provided-kafka-truststore.jks} -rfc -file {desired-kafka-cert-output.pem}`

- [**ssl.truststore.password**](http://kafka.apache.org/documentation/#adminclientconfigs_ssl.truststore.password)

  When providing a JKS-based truststore for validation of encryption certificates via the `ssl.truststore.location` configuration property above, you will generally need to provide a password to access the truststore file. This should be provided by your Kafka administrator at the same time they provide you with the truststore (or at least provide you with a method to acquire the truststore password). Truststores passwords are not required for PEM-based truststores, only JKS-based truststores.

- [**sasl.mechanism**](http://kafka.apache.org/documentation/#adminclientconfigs_sasl.mechanism)

  SASL stands for [_Simple Authentication and Security Layer_](https://en.wikipedia.org/wiki/Simple_Authentication_and_Security_Layer) and focuses on decoupling authentication mechanisms from application protocols - something that the majority of the early Internet-era applications and middleware did a very poor job of. As such, there are a number of pluggable mechanisms that can be included here to determine authentication between Kafka clients and brokers once the necessary communication protocol has been established with our previous setting.

  The most commonly encountered values here are:
  - `PLAIN` _(cleartext passwords, although they will be encrypted across the wire per **security.protocol** settings above)_
  - `SCRAM-SHA-512` _(modern Salted Challenge Response Authentication Mechanism)_
  - `GSSAPI` _(Kerberos-supported authentication and the default if not specified otherwise)_

- [**sasl.jaas.config**](http://kafka.apache.org/documentation/#adminclientconfigs_sasl.jaas.config)

  Java-based clients make use of the [Java Authentication and Authorization Service or JAAS](https://docs.oracle.com/javase/8/docs/technotes/guides/security/jaas/JAASRefGuide.html) framework that extends the pluggability of the SASL decoupling approach to the enterprise-secured Java programming language. This allows for flexible yet robust security configuration through JAAS configuration. All of this boils down to you needing to provide a string named `sasl.jaas.config` that contains three main components: (1) a class name, (2) a username, and (3) a password. The security components and plugins take it from there. This string is often provided by your Kafka administrator with all the components already filled in for you.

  Some commonly encountered `sasl.jaas.config` strings are:
  - `sasl.jaas.config = org.apache.kafka.common.security.plain.PlainLoginModule required username="{USERNAME}" password="{PASSWORD}";`
  - `sasl.jaas.config = org.apache.kafka.common.security.scram.ScramLoginModule required username="{USERNAME}" password="{PASSWORD}";`

## Common configuration options for the most popular offerings

### Strimzi / Red Hat AMQ Streams / IBM Event Streams

[Strimzi](https://strimzi.io/) is an open-source project which provides a Kubernetes Operator-focused way of deploying Apache Kafka clusters flexibly, repeatedly, and securely to any Kubernetes distribution. The enterprise-grade offerings of [Red Hat AMQ Streams](https://www.redhat.com/en/blog/getting-started-red-hat-amq-streams-operator) and [IBM Event Streams](https://ibm.github.io/event-streams/) are built on top of the underlying Strimzi Operator Custom Resources and take advantage of much of the same configuration specifications.

  Clients external to the Kubernetes or OpenShift cluster which Strimzi et al are deployed to often use the following configurations:
  - `bootstrap.servers={kafka-cluster-name}-kafka-bootstrap-{namespace}.{kubernetes-cluster-fully-qualified-domain-name}:443`
  - `security.protocol=SASL_SSL`
  - `sasl.mechanism=SCRAM-SHA-512`
  - `sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required username="{USERNAME}" password="{PASSWORD}";`
  - `ssl.truststore.location={/provided/to/you/by/the/kafka/administrator}`
  - `ssl.truststore.password={__provided_to_you_by_the_kafka_administrator__}`

  Clients internal to the Kubernetes or OpenShift cluster which Strimzi et al are deployed to often use the following configurations:
  - `bootstrap.servers={kafka-cluster-name}-kafka-bootstrap.{namespace}.svc.cluster.local:9093`
  - `security.protocol = SASL_PLAINTEXT` _(these clients do not require SSL-based encryption as they are local to the cluster)_
  - `sasl.mechanism = SCRAM-SHA-512`
  - `sasl.jaas.config = org.apache.kafka.common.security.scram.ScramLoginModule required username="{USERNAME}" password="{PASSWORD}";`

**NOTE:** Recent Strimzi versions have added support for OAuth-based authentication, beyond just SASL or mTLS-based authentication. We have not widely encountered this in the field yet, but will address it once we do.

For more information on the specifics of configuring [secure listeners on Strimzi-based Kafka clusters](https://strimzi.io/docs/operators/latest/using.html#assembly-securing-access-str), you will want to visit the official [Strimzi Docs](https://strimzi.io/docs/operators/latest/using.html). The examples in the documentation, as well as our demo scenario below, utilize three listeners with different security profiles _(PLAINTEXT vs SSL transport layers / no authentication vs SASL-based authentication vs mutual TLS authentication / internal vs external clients)_ based upon the expected clients connecting to them.

For information on configuring users and access credentials, visit the [Strimzi Docs](https://strimzi.io/docs/operators/latest/using.html) page for [User Authentication](https://strimzi.io/docs/operators/latest/using.html#con-securing-client-authentication-str) as the least-common-denominator between all three noted Strimzi variants. This method of user administration leverages the User Operator provided by Strimzi to manage users through Kubernetes Custom Resources.

### Confluent Platform

Confluent provides many configuration options and deployment targets for their Confluent Platform on-premises Kafka offering. Our team's main focus is Red Hat OpenShift-based deployments and as such, we focus on the capabilities made available to Confluent Platform through the [Confluent Operator](https://docs.confluent.io/operator/current/overview.html). As such, we focus [SASL and mTLS-based authentication mechanisms with optional TLS-encrypted traffic](https://docs.confluent.io/operator/current/co-authenticate.html).

Clients external to the Kubernetes or OpenShift cluster which Confluent Platform is deployed to most often use the following configurations:
  - `bootstrap.servers=kafka.{kubernetes-cluster-fully-qualified-domain-name}:443`
  - `sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username="{USERNAME}" password="{PASSWORD}";`
  - `security.protocol=SASL_SSL`
  - `sasl.mechanism=PLAIN`
  - `ssl.truststore.location={/provided/to/you/by/the/kafka/administrator}`
  - `ssl.truststore.password={__provided_to_you_by_the_kafka_administrator__}`

Clients internal to the Kubernetes or OpenShift cluster which Confluent Platform is deployed to most often use the following configurations:
  - `bootstrap.servers=kafka.{namespace}.svc.cluster.local:9071`
  - `sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username="{USERNAME}" password="{PASSWORD}";`
  - `security.protocol=SASL_PLAINTEXT`
  - `sasl.mechanism=PLAIN`

For information on configuring custom SASL users with the Confluent Operator, visit the [Confluent Docs](https://docs.confluent.io/operator/current/overview.html) page for [Authentication and Encryption](https://docs.confluent.io/operator/current/co-authenticate.html#add-custom-sasl-users)

### IBM Event Streams on Cloud

IBM provides a hosted, SaaS-based Kafka offering as part of IBM Cloud and it is available via the [IBM Cloud Catalog](https://cloud.ibm.com/catalog/services/event-streams). It comes pre-configured with everything you need to immediately get started working with Kafka applications, along with authentication that is directly integrated into the IBM Cloud IAM.

  - `bootstrap.servers=broker-0-{cluster-id}.kafka.{service-name}.eventstreams.cloud.ibm.com:9093,...,broker-5-{cluster-id}.kafka.{service-name}.eventstreams.cloud.ibm.com:9093` _(as the IBM Event Streams on Cloud does not leverage a bootstrap address for client connectivity)_
  - `sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username="{USERNAME}" password="{PASSWORD}";`
  - `security.protocol=SASL_SSL`
  - `sasl.mechanism=PLAIN`

For information on configuring users and access credentials, visit the [IBM Cloud Docs](https://cloud.ibm.com/docs) page for [Connecting to Event Streams](https://cloud.ibm.com/docs/EventStreams?topic=EventStreams-connecting) and [Kafka Clients](https://cloud.ibm.com/docs/EventStreams?topic=EventStreams-kafka_using#kafka_clients).

## Practical Use Case with Kafka binaries

Now we will walk through a straight-forward use case of deploying a vanilla open-source Strimzi cluster _(which remember uses the same underlying configuration as IBM Event Streams V10 and Red Hat AMQ Streams)_, creating a SCRAM-based user, creating a topic, and interacting with that topic through the CLI tools provided by the Kafka binaries.

The only parameters we will be configuring from the downstream client perspective are the ones we covered above. This quick tutorial assumes access to a Red Hat OpenShift 4.5+ cluster. If you do not have one, you can access one easily via the [IBM Open Labs](https://developer.ibm.com/openlabs/openshift). _(**NOTE:** You will want to avoid the OpenShift Playground clusters available on Katacoda for this exercise, as they use a more complex SSL certificate setup than the scope of this article is meant to address.)_

1. Access an OpenShift 4.5+ cluster via the OpenShift Console.

   ![1](/assets/2021-04-27-kafka-security-fundamentals-1/1.png)

1. Install the Strimzi Operator via the OpenShift Console to monitor all namespaces.

   1. Go to Operators on the left hand side menu and click on OperatorHub. Then, look up strimzi.
    ![2](/assets/2021-04-27-kafka-security-fundamentals-1/2.png)
   1. Click on the Strimzi tile and click on Install.
    ![3](/assets/2021-04-27-kafka-security-fundamentals-1/3.png)
   1. Make sure you are installing from the _stable_ channel and you are installing the operator in _All namespaces on the cluster_. Click Install.
    ![4](/assets/2021-04-27-kafka-security-fundamentals-1/4.png)
   1. After a couple of minutes, you should see the Strimzi operator successfully deployed and running.
    ![5](/assets/2021-04-27-kafka-security-fundamentals-1/5.png)

1. Create a new project named `kafka-security`.

   1. Go to Projects on the left hand side menu and click on Create Project.
    ![6](/assets/2021-04-27-kafka-security-fundamentals-1/6.png)
   1. Once the project is created, you should be presented with a dashboard for it.
    ![7](/assets/2021-04-27-kafka-security-fundamentals-1/7.png)

1. Create a Kafka cluster instance.
   1. Click on _Installed Operators_ under the Operators section in the left hand side menu. Click on the Strimzi operator.
    ![8](/assets/2021-04-27-kafka-security-fundamentals-1/8.png)
   1. Click on the Kafka tab and click on Create Kafka.
    ![9](/assets/2021-04-27-kafka-security-fundamentals-1/9.png)
   1. Switch to the YAML view.
    ![10](/assets/2021-04-27-kafka-security-fundamentals-1/10.png)
   1. Update the **listeners** section within the Spec with the following configuration

       ```yaml
       listeners:
         - name: plain
           port: 9092
           type: internal
           tls: false
         - name: tls
           port: 9093
           type: internal
           tls: true
         - name: external
           port: 9094
           type: route
           tls: true
           authentication:
             type: scram-sha-512
       ```

       The above _listeners_ configuration will create an internal non-secured listener on port `9092`, an internal TLS-secured listener on port `9093` and an external TLS-secured and scram-sha-512 authentication required on port `9094`. We will use this last port to access our Kafka cluster from outside of OpenShift. As a result, we will need a TLS certificate for the SSL connection plus a set of `scram-sha-512` credentials to get authenticated against.
    1. Make sure the **entityOperator** along with the **topicOperator** and the **userOperator** are also defined within the **spec** section. If they are not, add them.
      ![10-1](/assets/2021-04-27-kafka-security-fundamentals-1/10-1.png)
    1. Once you have updated the listeners section, click on Create. After a minute or so, you should see the Kafka Cluster ready.
      ![11](/assets/2021-04-27-kafka-security-fundamentals-1/11.png)

1. Create a Kafka topic via the Strimzi Operator.

   1. Click on the KafkaTopic tab and then click on Create KafkaTopic
    ![12](/assets/2021-04-27-kafka-security-fundamentals-1/12.png)
   1. Leave the defaults and click on Create.
    ![13](/assets/2021-04-27-kafka-security-fundamentals-1/13.png)
     _**NOTE:** Your Topic Name should stay `my-topic` for this simple example to align with the default ACLs for the KafkaUser you will create next._
   1. You should see your topic created.
    ![14](/assets/2021-04-27-kafka-security-fundamentals-1/14.png)

1. Create a KafkaUser via the Strimzi Operator

   1. Click on the KafkaUser tab and then click on Create KafkaUser
    ![15](/assets/2021-04-27-kafka-security-fundamentals-1/15.png)
   1. Click on Authentication and switch the authentication type to `scram-sha-512`. Click on Create.
    ![16](/assets/2021-04-27-kafka-security-fundamentals-1/16.png)
    This will create a Kafka user with a set of `scram-sha-512` credentials so that it can access the Kafka Cluster from outside of OpenShift using the external listener on port `9094` you created earlier.
   1. You should see the user created.
    ![17](/assets/2021-04-27-kafka-security-fundamentals-1/17.png)

1. Generate our configuration properties locally. On your terminal, execute the following commands.

   1. Download the Kafka binaries to your sandbox environment .
      ```bash
      curl https://downloads.apache.org/kafka/2.6.1/kafka_2.13-2.6.1.tgz -o kafka.tgz
      tar xvf kafka.tgz
      cd kafka_2.13-2.6.1
      ```

   1. Change project to `kafka-security`
      ```bash
      oc project kafka-security
      ```

   1. Export our configuration data to environment variables for easy reuse.
      ```bash
      export BOOTSTRAP="$(oc get route my-cluster-kafka-bootstrap -ojsonpath='{.spec.host}'):443"
      export CONFIG_FILE=local-config.properties
      ```

   1. Initialize local configuration file.
      ```bash
      rm -f ${CONFIG_FILE}
      ```

   1. Setup our three basic SASL/SSL properties.
      ```bash
      echo "sasl.jaas.config=$(oc get secret my-user -o json | jq -r '.data["sasl.jaas.config"]' | base64 -d -)" >> ${CONFIG_FILE}
      echo "sasl.mechanism=SCRAM-SHA-512" >> ${CONFIG_FILE}
      echo "security.protocol=SASL_SSL" >> ${CONFIG_FILE}
      ```
      Since we are going to try to work with the Kafka cluster we deployed earlier from outside of the OpenShift cluster it is deployed into, we need to do that through the external ssl-secured and scram-sha-512 authentication protected `9094` kafka listener we defined earlier when we defined the Kafka cluster we wanted to get deployed (the Strimzi operator will take care of creating the external access to that listener within the OpenShift cluster in the form of an OpenShift route). As a result, the `security.protocol` we need to set in the configuration file that the Kafka binaries scripts will later use is `SASL_SSL`. Then, we need to provide two things for this type of connection to a Kafka cluster: One is the SSL certificate for the secure communication and the other is the scram-sha-512 set of credentials. The first is provided through the `ssl.truststore` properties you can see in the next two steps. The other, the scram-sha-512 set of credentials, are provided through the `sasl` properties you can see above.

   2. Play the role of the Kafka Administrator and generate your Java Truststore artifact.
      ```bash
      oc extract secret/my-cluster-cluster-ca-cert --confirm --keys=ca.crt
      keytool -keystore cluster-ca.jks -import -file ca.crt -storepass my-cluster-password -noprompt
      rm -f ca.crt
      ```
      This is the certificate that will allow us to established the SSL communication to our Kafka cluster.

   3. Pass in the path to the Java Truststore artifact on the local filesystem (with absolute path).
      ```bash
      echo "ssl.truststore.location=$(pwd)/cluster-ca.jks" >> ${CONFIG_FILE}
      echo "ssl.truststore.password=my-cluster-password" >> ${CONFIG_FILE}
      ```
      Finally, we need to provide to the Kafka binaries scripts we are going to use next the location for the SSL certificate above.

1. View topic data via **`bin/kafka-topic.sh`** tool:
    ```bash
    bin/kafka-topics.sh --bootstrap-server ${BOOTSTRAP} \
        --command-config ${CONFIG_FILE} --list
    ```

1. Send data with the **`bin/kafka-console-producer.sh`** tool:
    ```bash
    bin/kafka-console-producer.sh --bootstrap-server ${BOOTSTRAP} \
        --producer.config ${CONFIG_FILE} --topic my-topic
    ```
    - **NOTE:** Use `Ctrl+C` when you are done sending data to exit the console producer.

1. Read data with the **`bin/kafka-console-consumer.sh`** tool:
    ```bash
    bin/kafka-console-consumer.sh --bootstrap-server ${BOOTSTRAP} \
        --consumer.config ${CONFIG_FILE} --topic my-topic --from-beginning
    ```
    - **NOTE:** Use `Ctrl+C` when you are done reading data to exit the console consumer.

  With this understanding of Kafka client security configuration, you can now seamlessly switch between Strimzi to IBM Event Streams to Red Hat AMQ Streams to Confluent Platform to IBM Event Streams on Cloud to [Lenses.io](https://lenses.io/) to [kafdrop](https://github.com/obsidiandynamics/kafdrop) to [kafkacat](https://github.com/edenhill/kafkacat) to [Kafka-provided binaries](https://github.com/apache/kafka/tree/trunk/bin) and beyond by following the same process!

## Conclusion
These same six settings will occur over and over again, from Producer Configs to Consumer Configs to Kafka Streams Configs to Kafka Connect Configs to Admin Client Configs to command-line configs. They are everywhere! Depending on your implementation language, platform, and tooling, there may be some assistance available in the SDKs, APIs, or CLIs to prevent the need of copying and pasting across the different Kafka component configurations, but once you get the configuration correct at the highest level, everything else in your event-driven applications seems much simpler to connect, debug, and validate at the lower levels!

I hope this whirlwind tour of practical Kafka security configuration options has been helpful and will become a handy reference for you to use going forward! Please let me know if you find this useful and would like to see more practical Kafka experiences documented here. In the meantime, head over to our [Event-Driven Reference Architecture](https://www.ibm.com/cloud/architecture/architectures/eventDrivenArchitecture/overview) in the [IBM Cloud Architecture Center](https://www.ibm.com/cloud/architecture/architectures) that has the latest and greatest collection of all things event-driven and Kafka-based!

> A special shoutout to Jesus Almaraz & Jerome Boyer for their reviews of, edits on, and polishing contributions to this article.

> Originally published via Medium at [https://rosowski.medium.com/kafka-security-fundamentals-the-rosetta-stone-to-your-event-streaming-infrastructure-518f49640db4](https://rosowski.medium.com/kafka-security-fundamentals-the-rosetta-stone-to-your-event-streaming-infrastructure-518f49640db4)
