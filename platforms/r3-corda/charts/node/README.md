# NODE

Following chart contains Kubernetes deployment, service, pvc ,configmap for deploying corda node. 
A node is JVM run-time with a unique network identity running the Corda software.

For more information read [corda node](https://docs.corda.net/releases/release-V3.3/key-concepts-node.html)


This chart has following structue:
```
  .
  ├── node
  │   ├── Chart.yaml
  │   ├── templates
  │   │   ├── configmap.yaml
  │   │   ├── deployment.yaml
  │   │   ├── _helpers.tpl
  │   │   ├── pvc.yaml
  │   │   └── service.yaml
  │   └── values.yaml
```

Type of files used:

```
charts.yaml       : A YAML file containing information about the chart
configmap.yaml    : ConfigMaps allow you to decouple configuration artifacts from image content to keep containerized applications portable
deployment.yaml   : A Deployment controller provides declarative updates for Pods and ReplicaSets.
_helpers.tpl      : A place to put template helpers that you can re-use throughout the chart
pvc.yaml          : A PersistentVolumeClaim (PVC) is a request for storage by a user.
service.yaml      : An abstract way to expose an application running on a set of Pods as a network service.
values.yaml       : This file contains the default values for a chart
```


## Running the chart

Pre-Requisite: Before deploying the chart please ensure you have Networkmap and Node's database up and running. 

- Deploy Networkmap & Node's Database by following steps from documentation 
- Create secrets for the node by following steps from documentation 
- Create a values-node.yaml for the chart with a minimum set of keys, for template references use values.yaml present in the respective chart
- Create aws cli script to transfer artmis folder (which gets created by corda node) to and from AWS s3  

Install the chart with:

```
helm install --set-file "awscliscript=<aws-cli script name>" -f ${PATH_TO_VALUES}/<node name>/values-node.yaml --set metadata.namespace=<node namespace> ${PATH_TO_HELMCHARTS}/node --name <helm name> --kube-context=<kube context> --namespace=<node namespace>

```

If you need to delete the chart use:

```
helm delete --purge <helm name> --kube-context=<kube context>
```

# Chart Functionalities

## deployment.yaml 

Contains following containers:

### Main Containers: 

1. corda-node: 	This container is used for running corda jar.  
  Tasks performed in this container	
- Setting up enviroment variables required for corda jar
- condition to check if artmis folder is recovered from s3 
- condtion to check if artmis folder retrived from s3 is empty or contains data 
- command to run corda jar, we are setting javax.net.ssl.keyStore as ${BASE_DIR}/certificates/sslkeystore.jks since keystore gets reset when using h2 ssl
- Delete empty artmis folder becacause it causes problem in starting corda node
- Import self signed tls certificate (if used) of doorman and networkmap, since java only trusts certificate signed by well known CA  
- clean network-arameters on every restart
- import self signed tls certificate of H2, since java only trusts certificate signed by well known CA 

2. corda-logs:  This container is is used for printing corda logs  
  Tasks performed in this container
- Loop to check if log file is generated by corda and keep on printing log file if it is generated by corda

3. lifecycle-artemis: This container is is used for printing corda logs  
  Tasks performed in this container
- Download artemis folder from s3 when starting
- Upload artemis folder to s3 when deleting 

4. corda-web: Following container is formed if webserver is enabled and number of container created depends upon no of websever mentioned by user in values.yaml file  
  Tasks performed in this container
- copy app.jar from image to mounted path       
- perform health check if db is up and running before starting corda node
- To run smart contract jar with springboot if it is provided 

### Init-containers:

1. init-nodeconf: This container is used for creating node.conf which is used by corda node.  
  For more details on how to make node.conf read [node configuration](https://docs.corda.net/releases/release-V3.3/corda-configuration-file.html)  
  Tasks performed in this container  
- delete previously created node.conf, and create a new node.conf
- set env to get secrets from vault
- save keyStorePassword & trustStorePassword from vault
- save dataSourceUserPassword from vault
- create node.conf according to values given by users using values.yaml

2. init-downloadcordajars: This container is used for downloading corda jar  
   For more details on read [corda v3.3](https://github.com/corda/corda/tree/release-V3.3)  
   Tasks performed in this container
-  Download corda jar if it doesnt exist

3. init-certificates:  This container is used for downloading certficate from vault  	
   For more details on read [Network permissioning](https://docs.corda.net/releases/release-V3.3/permissioning.html)
   Tasks performed in this container
- setting up env to get secrets from vault
- get nodekeystore.jks from vault
- get sslkeystore.jks from vault
- get truststore.jks from vault
- get network-map-truststore.jks from vault
- when using h2-ssl with private certificate, download the certificate  (To import in ca cert in main corda container)
- when using doorman and networkmap in TLS: true, and using private certificate then download certificate(To import in ca cert in main corda container)
- when using custom sslKeystore while setting in node.conf
- To download jars from git repo, download private key (corresponding public key to be added in git repo)
- get aws access key id and secret access key, it is used for accessing AWS s3 for artmis folder 

4. init-downloadcordapps: This container is used for downloading corda jar from git repo or any links provided in values.yaml  
  For more details on read [cordapp](https://docs.corda.r3.com/releases/3.3/cordapp-build-systems.html)
  Tasks performed in this container
- crearting cordapps dir in volume to keep jars
- Deleting cordapps dir to get rid of old jars
- removing /tmp/corda-jars and /tmp/downloaded-jars to start fresh
- Setting up env for git clone
- Git repository clone for cordapps
- copy the jars from repository to node volume 
- Download official corda provided jars using wget
- copy the jars from repository to web volume
- remove private key & tmp dir dir created 
- Print total jars present in node dir
NOTE:- Update git repository url and branch were jars are present in values.

5. init-healthcheck: This container is used for performing health check  
  Tasks performed in this container
- perform health check if db is up and running before starting corda node

6. init-webproperties: Following container is formed if webserver is enabled and number of container created depends upon no of websever mentioned by user in values.yaml file . It is used for creating web properties by values passed from values.yaml and vault.  
  Tasks performed in this container
- delete previously created app.properties, and create a new app.properties
- set env to get secrets from vault
- retrive keystore passwords and rpc user passwords from rpc
- create app.properties using values specified by user

## configmap.yaml

Contains following data:

1. awscliscript.sh: <enter> Script for uploading and downloading corda's artmis folder to and from AWS s3.  
   Script to be passed as a file in helm command using --set-file

## service.yaml

Contains port specifications for:

1. p2p communication among corda node
2. rpc communication between corda node and webserver
3. rpc admin communication
4. communication with springboot web  (added only if webserver is enabled and number is based on values specified by user in values.yaml)

## pvc.yaml

Contains specifications for:

1. To create pvc used by node
2. To create pvc used for each webserver deployed (added only if webserver is enabled and number is based on values specified by user in values.yaml)
NOTE:- On helm deletion of node chart data stored in volume will get lost due to deletion of persistence volume claim.
```
helm upgrade --install --set-file "awscliscript=<aws-cli script name>" -f ${PATH_TO_VALUES}/<node name>/values-node.yaml --set metadata.namespace=<node namespace> ${PATH_TO_HELMCHARTS}/node --name <helm name> --kube-context=<kube context> --namespace=<node namespace>
```