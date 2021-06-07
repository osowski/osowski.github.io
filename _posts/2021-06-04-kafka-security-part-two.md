---
title: 'Kafka Security Fundamentals — adding TLS to your event-driven utility belt!'
date: 2021-06-04T12:30:00-05:00
author: Rick Osowski
layout: post
permalink: /2021/06/kafka-security-fundamentals-2/
canonical_url: 'https://rosowski.medium.com/kafka-security-fundamentals-adding-tls-to-your-event-driven-utility-belt-432307f4ff62'
categories:
  - Technology
tags:
  - Apache Kafka
  - Event-Driven Architecture
  - Event Streaming
---


# Apache Kafka security - adding TLS to your event-driven utility belt!

By [Rick Osowski](https://medium.com/@rosowski)

Published via Medium at [https://rosowski.medium.com/kafka-security-fundamentals-adding-tls-to-your-event-driven-utility-belt-432307f4ff62](https://rosowski.medium.com/kafka-security-fundamentals-adding-tls-to-your-event-driven-utility-belt-432307f4ff62)

## Introduction

  Understanding [Apache Kafka](https://kafka.apache.org/) security is much like discovering the Rosetta Stone to the entire event-streaming landscape. In my [previous blog post](https://rosowski.medium.com/kafka-security-fundamentals-the-rosetta-stone-to-your-event-streaming-infrastructure-518f49640db4), I covered how understanding a few common core configurations provides you the ability to connect any Kafka-based, event-driven application to dozens of Kafka offerings with relatively few changes. However, the previous blog post focused on username and password-based authentication. This blog post will cover those same core configurations but from the perspective of SSL Certificate-based authentication. Or as it is more commonly known, Mutual TLS (mTLS)-based authentication.

  As a caveat, this blog post will continue to cover the most important Kafka security configuration options from a Kafka client perspective _(based on [my team's experiences](https://ibm-cloud-architecture.github.io/refarch-eda/) so far)_ to allow you to pull this knowledge from your utility belt like a certain dark knight when it comes to any Kafka offering or security model.

  This is a deeper dive into Apache Kafka and its capabilities, so I will not be covering any introductory Kafka material here. You can head over to [kafka.apache.org](https://kafka.apache.org) for all the foundational information necessary or check out my friend Whitney's [What is Kafka?](https://www.youtube.com/watch?v=aj9CDZm0Glc&t=1s) video to learn everything you need to get started in under 10 minutes. Once you've read through everything here, if you still have questions or want to dig in further, the [Security](http://kafka.apache.org/documentation/#security) section of the official Kafka documentation covers everything in depth with a great deal of clarity - so that would be my first recommendation if you read through here to find that none of the common configurations below match your expected environment setups!

  Before we jump in, there is one remaining caveat from our last blog post to address:

  - There are many, many ways to configure Kafka security from a Kafka administrator point-of-view. The focus of this short blog series is to more effectively comprehend Kafka security as a Kafka client (producer or consumer) and not a Kafka administrator (operations / provider). For a deeper understanding of practical Kafka operational security from an adminstrator's point of view, you can reference Confluent's [Security Overview](https://docs.confluent.io/platform/current/security/general-overview.html) for a much deeper understanding on how on to securely deploy and maintain Kafka well into the future!

  Without any further ado, let's get started!

## SSL Certificate Rewind

<iframe width="560" height="315" src="https://www.youtube.com/embed/T4Df5_cojAs" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

The [above video](https://www.youtube.com/watch?v=T4Df5_cojAs) is a great quick introduction (or refresher) for [SSL/TLS certificates](https://en.wikipedia.org/wiki/Public_key_certificate) and the underlying assumptions and properties the majority of our digital security is founded upon. Borrowing the metaphor the speaker in the video uses, the two major assumptions of SSL security (and all [public-key cryptography](https://en.wikipedia.org/wiki/Public-key_cryptography)) are the following:

   1. Any message encrypted with Bob's public key can only be decrypted with Bob's private key
   2. Anyone with access to Alice's public key can verify that a message could only have been created by someone with access to Alice's private key.

For our use case, these two assumptions serve the exact function we need to perform both encryption and authentication by identity in our Kafka ecosystem. As such, the Kafka cluster components and your Kafka clients will need to share the necessary SSL/TLS certificates and keys between each other to communicate effectively, in a secured and controlled manner. This can be both complex and utterly confusing at the same time. But hopefully, after watching the video above and reading through the rest of this blog post, you will have a few more tools in your tool belt to tackle these security requirements.

With all that said, the minimum set of artifacts required for a Kafka client to authenticate with a Kafka cluster utilizing mTLS authentication is:
 - SSL Trust Store, containing:
    - the Certificate Authority of the Kafka cluster (its public key) _(or an intermediate certificate used for signing purposes that the Kafka cluster will trust)_
 - SSL Trust Store password
 - SSL Key Store, containing:
   - the Certificate Authority of the Kafka cluster (its public key) _(or an intermediate certificate used for signing purposes that the Kafka cluster will trust)_
   - the client certificate key-pair, signed by the Certificate Authority (both the public and private keys of the client)
 - SSL Key Store password

## Kafka Settings

- [**bootstrap.servers**](http://kafka.apache.org/documentation/#adminclientconfigs_bootstrap.servers)

  This is the most important setting as it tells Kafka clients where to look to talk to your applications. It's context, value, and explanation is unchanged from the previous blog post. Sometimes this will be a string that contains just a single `hostname:port` combination, while other times it will be a comma-separated list of `hostname:port` combinations. Depending upon the Kafka offering you are connecting to, as well as how and where it is deployed, you will either be providing a singular `bootstrap` address or the entire collection of Kafka brokers in the same string. The `bootstrap` endpoint is a functional intermediary that Kafka offerings provide to allow for simpler networking in cases of expected fluctuations in the number of Kafka brokers, as it is a single URL that will return all the active brokers in the cluster to your clients automatically.

- [**security.protocol**](http://kafka.apache.org/documentation/#adminclientconfigs_security.protocol)

  This is the next most important security setting as it tells the clients how to communicate with the brokers. If you've ever had HTTPS issues while using a web browser, this is the setting that would give Kafka clients SSL issues while attempting to connect to Kafka brokers! If you are receiving errors in your application while attempting to connect to Kafka with references to a `broker (-1)`, you will usually need to revist this setting _(i.e. you should be using `SASL_SSL` but you configured `PLAINTEXT`, etc.)_.

  The previous blog post used the values of `SASL_PLAINTEXT` or `SASL_SSL` to tell Kafka we wanted to communicate with SASL-based usernames and passwords. However, if you need to authenticate with mTLS certificates instead, your value for this configuration option will always be **`SSL`**.

  The valid values are:
  - `PLAINTEXT` _(using PLAINTEXT transport layer & no authentication - default value)_
  - `SSL` _(using SSL transport layer & certificate-based authentication)_
  - `SASL_PLAINTEXT` _(using PLAINTEXT transport layer & SASL-based authentication)_
  - `SASL_SSL` _(using SSL transport layer & SASL-based authentication)_

- [**ssl.truststore.location**](http://kafka.apache.org/documentation/#adminclientconfigs_ssl.truststore.location)

  To allow for encryption of communication between Kafka brokers and clients, as specified by our `security.protocol` setting above, we need to provide our Kafka clients with the location of a trusted Certificate Authority-based certificate. This file is often provided by the Kafka administrator and is generally unique to the specific Kafka cluster deployment. It is outside the scope of this blog post to determine how you provide access to this file for your individual runtimes, but it must exist inside the runtime container somewhere.

  For Java-based clients, this file is usually in the JKS format by default. However, in recent Kafka releases, P12 support has been added for Java clients as well. For Python or NodeJS-based clients, this file should be in either a PEM or P12 format. To extract a PEM-based certificate from a JKS-based truststore, you can use the following command:
  `keytool -exportcert -keypass {truststore-password} -keystore {provided-kafka-truststore.jks} -rfc -file {desired-kafka-cert-output.pem}`

- [**ssl.truststore.password**](http://kafka.apache.org/documentation/#adminclientconfigs_ssl.truststore.password)

  When providing a JKS or P12-based truststore for validation of encryption certificates via the `ssl.truststore.location` configuration property above, you will generally need to provide a password to access the truststore file. This should be provided by your Kafka administrator at the same time they provide you with the truststore (or at least provide you with a method to acquire the truststore password). Truststores passwords are not required for PEM-based truststores, only JKS-based truststores.

- [**ssl.keystore.location**](http://kafka.apache.org/documentation/#adminclientconfigs_ssl.keystore.location)

  This setting is where the Kafka client provides its certificate-based authentication credentials to the Kafka brokers, as specified by our `security.protocol` setting as `SSL`. We need to provide our Kafka clients with the location of a unique identifying certificate that is signed by the overall Cluster CA. This file is often provided by the Kafka administrator or generated as part of initial client or user configuration. It is outside the scope of this blog post to determine how you provide access to this file for your individual runtimes, but it must exist inside the runtime container somewhere.

  For Java-based clients, this file is usually in the JKS format by default. For Python or NodeJS-based clients, this file should be in either a PEM or P12 format. To extract a PEM-based certificate from a JKS-based truststore, you can use the following command:
  `keytool -exportcert -keypass {keystore-password} -keystore {provided-client-keystore.jks} -rfc -file {desired-client-cert-output.pem}`

- [**ssl.keystore.password**](http://kafka.apache.org/documentation/#adminclientconfigs_ssl.keystore.password)

  Similar to `ssl.truststore.password`, when providing a JKS or P12-based keystore for client authentication purposes via the `ssl.keystore.location` configuration property above, you will generally need to provide a password to access the associated keystore file. This can be provided by your Kafka administrator at the same time they provide you with the CA truststore (or at least provide you with a method to acquire any additional client keystore/key passwords). Keystore passwords are not required for PEM-based truststores, only JKS or P12-based truststores.

- [**ssl.key.password**](http://kafka.apache.org/documentation/#adminclientconfigs_ssl.key.password)

  This configuration property contains the value of the password to the private key that is in the keystore file _(or the PEM key if specified explicitly in `ssl.keystore.key`, but that is an alternate configuration setting we are not covering in this blog post.)_ This value is most often the same as `ssl.keystore.password` above and is explicitly required when clients are to perform two-way authentication, which is our use case here. Similar to the other `ssl.keystore.*` settings, this can be provided by your Kafka administrator at the same time they provide you with the CA truststore (or at least provide you with a method to acquire any additional client keystore/key passwords).

## Common configuration options for the most popular offerings

  As you will see in the sections below, the differences between Kafka offerings when using mTLS-based authentication is minimal. All the technical detail comes in how you are generating the SSL certificates that you are providing to your clients. In other words, the devil is in the details. These sections will still cover the specific settings necessary for each offering type, with additional reference material where appropriate for help in understanding SSL certificate generation for each offering. Comprehensive SSL certificate management for each Kafka offering is outside the scope of this blog post.

### Strimzi / Red Hat AMQ Streams / IBM Event Streams

  [Strimzi](https://strimzi.io/) is an open-source project which provides a Kubernetes Operator-focused way of deploying Apache Kafka clusters flexibly, repeatedly, and securely to any Kubernetes distribution. The enterprise-grade offerings of [Red Hat AMQ Streams](https://www.redhat.com/en/blog/getting-started-red-hat-amq-streams-operator) and [IBM Event Streams](https://ibm.github.io/event-streams/) are built on top of the underlying Strimzi Operator Custom Resources and take advantage of much of the same configuration specifications.

  The power of the Operator model that the Strimzi variants bring to Kafka is the automated maintenance and operational detail that is otherwise necessary to do explicitly by hand or custom automation. Strimzi provides a UserOperator and a TopicOperator subcomponent for administration of those contextual objects. In the example to be covered later on in this blog post, we leverage the UserOperator to automatically generate signed SSL certificates for a new user which are based off of the internal Certificate Authority contained in the Strimzi Kafka deployment. There is no need for manual SSL certificate configuration! For information on configuring users and access credentials this way, visit the [Strimzi Docs](https://strimzi.io/docs/operators/latest/using.html) page for [User Authentication](https://strimzi.io/docs/operators/latest/using.html#con-securing-client-authentication-str) as the least-common-denominator between all three noted Strimzi variants.

  External clients connecting to the Kubernetes or OpenShift cluster which Strimzi et al are deployed to often use the following configurations when requiring two-way SSL/mTLS authentication:
  - `bootstrap.servers={kafka-cluster-name}-kafka-bootstrap-{namespace}.{kubernetes-cluster-fully-qualified-domain-name}:443`
  - `security.protocol=SSL`
  - `ssl.truststore.location={/provided/to/you/by/kafka/administrator}`
  - `ssl.truststore.password={__provided_to_you_by_kafka_administrator__}`
  - `ssl.keystore.location={/extracted/from/generated/kafka/user/secret}/user.p12`
  - `ssl.keystore.password={__extracted_from_generated_kafka_user_secret_with_key=user.password__}`
  - `ssl.key.password={__extracted_from_generated_kafka_user_secret_with_key=user.password__}`

  Clients internal to the Kubernetes or OpenShift cluster which Strimzi et al are deployed to often use the following configurations:
  - `bootstrap.servers={kafka-cluster-name}-kafka-bootstrap.{namespace}.svc.cluster.local:9093`
  - `security.protocol=SSL`
  - `ssl.truststore.location={/provided/to/you/by/kafka/administrator}`
  - `ssl.truststore.password={__provided_to_you_by_kafka_administrator__}`
  - `ssl.keystore.location={/extracted/from/generated/kafka/user/secret}/user.p12`
  - `ssl.keystore.password={__extracted_from_generated_kafka_user_secret_with_key=user.password__}`
  - `ssl.key.password={__extracted_from_generated_kafka_user_secret_with_key=user.password__}`

  **NOTE:** Recent Strimzi versions have added support for OAuth-based authentication, beyond just SASL or mTLS-based authentication. We have not widely encountered this in the field yet, but will address it once we do.

  For more information on the specifics of configuring [secure listeners on Strimzi-based Kafka clusters](https://strimzi.io/docs/operators/latest/using.html#assembly-securing-access-str), you will want to visit the official [Strimzi Docs](https://strimzi.io/docs/operators/latest/using.html). The examples in the documentation, as well as our demo scenario below, utilize three listeners with different security profiles _(PLAINTEXT vs SSL transport layers / no authentication vs SASL-based authentication vs mutual TLS authentication / internal vs external clients)_ based upon the expected clients connecting to them.

### Confluent Platform

  Confluent provides many configuration options and deployment targets for their Confluent Platform on-premises Kafka offering. Our team's main focus is Red Hat OpenShift-based deployments and we focus on the capabilities made available to Confluent Platform through the [Confluent Operator](https://docs.confluent.io/operator/current/overview.html). As such, we focus on [SASL and mTLS-based authentication mechanisms with optional TLS-encrypted traffic](https://docs.confluent.io/operator/current/co-authenticate.html).

  >Confluent has recently released an updated Kubernetes Operator, called [Confluent for Kubernetes](https://docs.confluent.io/operator/current/overview.html). This blog post will speak to the previous version of the Operator, [Confluent Operator 1](https://docs.confluent.io/operator/1.7.0/overview.html), to align with the previous blog post in this series. Minimal technical changes are expected between the two Operator versions as they pertain to the use cases discussed here.

  For a deeper dive into the extent of SSL and Mutual TLS security inside of Confluent Platform, you can refer to the robust [Security Tutorial](https://docs.confluent.io/platform/current/security/security_tutorial.html) in the Confluent Docs and specifically the section on [Creating SSL Keys and Certificates](https://docs.confluent.io/platform/current/security/security_tutorial.html#creating-ssl-keys-and-certificates). For a more streamlined approach to what is minimally required for a valid SSL client certificate in a Confluent Platform environment, you can reference the [Confluent Platform Security Tools](https://github.com/confluentinc/confluent-platform-security-tools) GitHub repository for scripts that automate the creation of the necessary keystores and truststores Kafka clients need.

  > For an ultra-streamlined version of the Confluent Platform Security Tools scripts when you already have a cluster CA certificate and key, you can refer to this [gist](https://gist.github.com/osowski/6abf268e9d7ab521481cc35523bc50f6) I adapted from the Confluent Platform Security Tools scripts.

  Clients external to the Kubernetes or OpenShift cluster which Confluent Platform is deployed to most often use the following configurations:
  - `bootstrap.servers=kafka.{kubernetes-cluster-fully-qualified-domain-name}:443`
  - `security.protocol=SSL`
  - `ssl.truststore.location={/provided/to/you/by/kafka/administrator}`
  - `ssl.truststore.password={__provided_to_you_by_kafka_administrator__}`
  - `ssl.keystore.location={/generated/in/coordination/with/kafka/adminstrator}`
  - `ssl.keystore.password={__generated_in_coordination_with_kafka_administrator__}`
  - `ssl.key.password={__generated_in_coordination_with_kafka_administrator__}`

  Clients internal to the Kubernetes or OpenShift cluster which Confluent Platform is deployed to most often use the following configurations:
  - `bootstrap.servers=kafka.{namespace}.svc.cluster.local:9071`
  - `security.protocol=SSL`
  - `ssl.truststore.location={/provided/to/you/by/kafka/administrator}`
  - `ssl.truststore.password={__provided_to_you_by_kafka_administrator__}`
  - `ssl.keystore.location={/generated/in/coordination/with/kafka/adminstrator}`
  - `ssl.keystore.password={__generated_in_coordination_with_kafka_administrator__}`
  - `ssl.key.password={__generated_in_coordination_with_kafka_administrator__}`

### IBM Event Streams on Cloud

  IBM provides a hosted, SaaS-based Kafka offering as part of IBM Cloud and it is available via the [IBM Cloud Catalog](https://cloud.ibm.com/catalog/services/event-streams). It comes pre-configured with everything you need to immediately get started working with Kafka applications, along with authentication that is directly integrated into the IBM Cloud IAM. As such, it does not support SSL-based authentication for clients at this time.

## Practical Use Case with Kafka binaries

  Now we will walk through a straight-forward use case of deploying a vanilla open-source Strimzi cluster _(which remember uses the same underlying configuration as IBM Event Streams V10 and Red Hat AMQ Streams)_, creating a TLS-based user, creating a topic, and interacting with that topic through the CLI tools provided by the Kafka binaries.

  The only parameters we will be configuring from the downstream client perspective are the ones we covered above. This quick tutorial assumes access to a Red Hat OpenShift 4.5+ cluster. If you do not have one, you can access one easily via the [IBM Open Labs](https://developer.ibm.com/openlabs/openshift). _(**NOTE:** You will want to avoid the OpenShift Playground clusters available on Katacoda for this exercise, as they use a more complex SSL certificate setup than the scope of this article is meant to address.)_

1. Access an OpenShift 4.5+ cluster via the OpenShift Console.

   ![1](/assets/2021-06-04-kafka-security-fundamentals-2/1.png)

1. Install the Strimzi Operator via the OpenShift Console to monitor all namespaces.

   1. Go to Operators on the left hand side menu and click on OperatorHub. Then, look up strimzi.
    ![2](/assets/2021-06-04-kafka-security-fundamentals-2/2.png)
   1. Click on the Strimzi tile and click on Install.
    ![3](/assets/2021-06-04-kafka-security-fundamentals-2/3.png)
   1. Make sure you are installing from the _stable_ channel and you are installing the operator in _All namespaces on the cluster_. Click Install.
    ![4](/assets/2021-06-04-kafka-security-fundamentals-2/4.png)
   1. After a couple of minutes, you should see the Strimzi operator successfully deployed and running.
    ![5](/assets/2021-06-04-kafka-security-fundamentals-2/5.png)

1. Create a new project named `kafka-security`.

   1. Go to Projects on the left hand side menu and click on Create Project.
    ![6](/assets/2021-06-04-kafka-security-fundamentals-2/6.png)
   1. Once the project is created, you should be presented with a dashboard for it.
    ![7](/assets/2021-06-04-kafka-security-fundamentals-2/7.png)

1. Create a Kafka cluster instance.
   1. Click on _Installed Operators_ under the Operators section in the left hand side menu. Click on the Strimzi operator.
    ![8](/assets/2021-06-04-kafka-security-fundamentals-2/8.png)
   1. Click on the Kafka tab and click on Create Kafka.
    ![9](/assets/2021-06-04-kafka-security-fundamentals-2/9.png)
   1. Switch to the YAML view.
    ![10](/assets/2021-06-04-kafka-security-fundamentals-2/10.png)
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
             type: tls
       ```
       The above _listeners_ configuration will create an internal non-secured listener on port `9092`, an internal TLS-secured listener on port `9093`, and an external TLS-secured listener on port `9094`. We will use this last port to access our Kafka cluster from outside of OpenShift. As a result, we need a TLS certificate for the SSL encryption and authentication.
    1. Make sure the **entityOperator** along with the **topicOperator** and the **userOperator** are also defined within the **spec** section. If they are not, add them.
      ![10-1](/assets/2021-06-04-kafka-security-fundamentals-2/10-1.png)
    1. Once you have updated the listeners section, click on Create. After a minute or so, you should see the Kafka Cluster ready.
      ![11](/assets/2021-06-04-kafka-security-fundamentals-2/11.png)

1. Create a Kafka topic via the Strimzi Operator.

   1. Click on the KafkaTopic tab and then click on Create KafkaTopic
    ![12](/assets/2021-06-04-kafka-security-fundamentals-2/12.png)
   1. Leave the defaults and click on Create.
    ![13](/assets/2021-06-04-kafka-security-fundamentals-2/13.png)
     _**NOTE:** Your Topic Name should stay `my-topic` for this simple example to align with the default ACLs for the KafkaUser you will create next._
   1. You should see your topic created.
    ![14](/assets/2021-06-04-kafka-security-fundamentals-2/14.png)

1. Create a KafkaUser via the Strimzi Operator

   1. Click on the KafkaUser tab and then click on Create KafkaUser
    ![15](/assets/2021-06-04-kafka-security-fundamentals-2/15.png)
   1. Click on Authentication and ensure the authentication type is set to `tls`. Click on Create.
    ![16](/assets/2021-06-04-kafka-security-fundamentals-2/16.png)
    This will create a Kafka user with a set of `tls` certificates so that it can access the Kafka Cluster from outside of OpenShift using the external listener on port `9094` you created earlier.
   1. You should see the user created.
    ![17](/assets/2021-06-04-kafka-security-fundamentals-2/17.png)

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
      export CONFIG_FILE=local-config-tls.properties
      export CACERT_DIR=cacert-blog
      export KAFKAUSER_DIR=my-user-blog
      ```

   1. Initialize local configuration file with the SSL security protocol.
      ```bash
      rm -f ${CONFIG_FILE}
      echo "security.protocol=SSL" >> ${CONFIG_FILE}
      ```
      Since we are working with the Kafka cluster we deployed earlier from outside of the OpenShift cluster it is deployed on, we need to do that through the external TLS-secured `9094` listener. The Strimzi Operator will take care of creating the external access to that listener within the OpenShift cluster in the form of an OpenShift Route. As a result, the `security.protocol` we need to set in the configuration file that the Kafka binaries scripts will later use is `SSL`.

   1. Play the role of the Kafka Administrator, extract your Truststore artifact, and pass in the path to the Java Truststore artifact on the local filesystem (with absolute path).
      ```bash
      rm -rf ${CACERT_DIR}
      mkdir ${CACERT_DIR}
      oc extract secret/my-cluster-cluster-ca-cert --to=./${CACERT_DIR}
      echo "ssl.truststore.location=$(pwd)/${CACERT_DIR}/ca.p12" >> ${CONFIG_FILE}
      echo "ssl.truststore.password=$(cat ${CACERT_DIR}/ca.password)" >> ${CONFIG_FILE}
      ```
      This certificate will allow us to establish SSL-encrypted outbound communication to our Kafka cluster.

    1. As a prospective user of the Kafka cluster, the Strimzi Operator has created a Kubernetes Secret containing all the necessary SSL certificate components you need to authenticate and communicate with the Kafka cluster securely. Extract your Keystore artifact and pass in its absolute path on the local filesystem.
       ```bash
       rm -rf ${KAFKAUSER_DIR}
       mkdir ${KAFKAUSER_DIR}
       oc extract secret/my-user --to=./${KAFKAUSER_DIR}
       echo "ssl.keystore.location=$(pwd)/${KAFKAUSER_DIR}/user.p12" >> ${CONFIG_FILE}
       echo "ssl.keystore.password=$(cat ${KAFKAUSER_DIR}/user.password)" >> ${CONFIG_FILE}
       echo "ssl.key.password=$(cat ${KAFKAUSER_DIR}/user.password)" >> ${CONFIG_FILE}
       ```
       This certificate will allow us to authenticate properly with the Kafka cluster and establish our client's cryptographically-secured identity.

       > The files extracted from the `my-user` Secret are the necessary SSL certificates and passwords that are automatically generated for us by the Strimzi Operator. There is no need for manual SSL certificate management when using the Strimzi Operator. When using the Confluent Platform, you would need to perform the steps documented in the [Confluent Platform Security Tools scripts](https://github.com/confluentinc/confluent-platform-security-tools) or my [streamlined gist](https://gist.github.com/osowski/6abf268e9d7ab521481cc35523bc50f6) to get those same files.

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

  These same handful of settings will occur over and over again, from Producer Configs to Consumer Configs to Kafka Streams Configs to Kafka Connect Configs to Admin Client Configs to command-line configs. They are everywhere! Depending on your implementation language, platform, and tooling, there may be some assistance available in the SDKs, APIs, or CLIs to prevent the need of copying and pasting across the different Kafka component configurations, but once you get the configuration correct at the highest level, everything else in your event-driven applications seems much simpler to connect, debug, and validate at the lower levels!

  Now with the addition of SSL certificate-based mutual authentication (mTLS), you are able to build, configure, and deploy all types of Kafka clients that will easily adapt to security models and infrastructure requirements - whether that is remote credential management or a cloud-provider-based key management system to handle and rotate your certificates regularly.

  I hope this whirlwind tour of practical Kafka security configuration options has been helpful and will become a handy reference for you to use going forward! Please let me know if you find this useful and would like to see more practical Kafka experiences documented here. In the meantime, head over to our [Event-Driven Reference Architecture](https://www.ibm.com/cloud/architecture/architectures/eventDrivenArchitecture/overview) in the [IBM Cloud Architecture Center](https://www.ibm.com/cloud/architecture/architectures) that has the latest and greatest collection of all things event-driven and Kafka-based!

> A special shoutout to Jesus Almaraz & Jerome Boyer for their reviews of, edits on, and polishing contributions to this article.

> Originally published via Medium at [https://rosowski.medium.com/kafka-security-fundamentals-adding-tls-to-your-event-driven-utility-belt-432307f4ff62](https://rosowski.medium.com/kafka-security-fundamentals-adding-tls-to-your-event-driven-utility-belt-432307f4ff62)
