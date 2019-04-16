Table of Contents
=================

   * [GCP Deployment](#webinar-deployment)
      * [Before starting](#before-starting)
         * [Pre-requisites](#pre-requisites)
         * [NGINX Ingress controller](#nginx-ingress-controller)
		 * [Certificate manager](#certificate-manager)
      * [Creating](#creating)
         * [Fabric CA](#fabric-ca)
         * [Crypto materials](#crypto-material)
         * [Kafka for Ordering service](#kafka-for-ordering-service)
         * [Fabric Orderers](#fabric-orderer)
         * [Fabric Peers](#fabric-peer)
      * [Deleting](#deleting)
      * [Extra resources](#extra-resources)
         * [Repositories](#repositories)
         * [Courses](#courses)
         * [FAQ](#faq)

# Webinar deployment

> Original note: The content of these instructions has changed since the webinar, following improvements in the hlf-ca, hlf-ord and hlf-peer charts. You also may wish to look at the newer https://github.com/aidtechnology/hgf-k8s-workshop repository and/or try your hands at https://github.com/aidtechnology/nephos, which helps automate the deployment process.

## Before starting

### Pre-requisites

Before running this tutorial you will need:

1) A Kubernetes (K8S) cluster on GCP - 4 nodes (Name: blockchain; Location type: Zonal; Zone: us-west1-a; Master version: 1.12.6-gke.10; Number of nodes: 4, Machine type: 2 vCPUs, 7.5 GB Memory)
2) Have gcloud SDK and kubectl installed locally (command line) and set K8S context: 
	Click Connect for K8S cluster and get command line access. Execute locally to set K8S context
3) Install Helm locally and Tiller on K8S: 

	```
	git clone https://github.com/yonghuigit/lf-k8s-hlf-webinar.git

	cd lf-k8s-hlf-webinar

	bash add_helm.sh
	```
	
4) Install K8S dashboard (optional):

	```
	kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/master/aio/deploy/recommended/kubernetes-dashboard.yaml

	kubectl create clusterrolebinding kubernetes-dashboard --clusterrole=cluster-admin --serviceaccount=kube-system:kubernetes-dashboard

	TOKEN=$(kubectl -n kube-system describe secret kubernetes-dashboard-token | awk '$1=="token:"{print $2}')

	echo $TOKEN

	kubectl proxy &
	```

5) Open browser to access K8S dashboard and log in with Token (from above, Optional))

	http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/

6) A domain name for your components (e.g. the Certificate Authority). You can obtain one for free or $1.00 at many Domain Name Registrars. Set it in ./helm_values/ca_values.yaml

#### NGINX Ingress controller

Install the ingress controller by running this command. Note this will create a load balancer in GCP for the ingress (K8S Service of type: LoadBalancer). Get the External endpoint IP and set it in DNS for the domain name above. Check with ping.

    helm install stable/nginx-ingress -n nginx-ingress --namespace ingress-controller

#### Certificate manager

Install the certificate manager, to ensure you can auto-generate the TLS certificates

    helm install stable/cert-manager -n cert-manager --namespace cert-manager --version v0.5.2

Then we need to add the Staging and Production cluster issuers

    kubectl create -f ./extra/certManagerCI_staging.yaml

    kubectl create -f ./extra/certManagerCI_production.yaml

## Creating

### Fabric CA

#### Installing

Install the Fabric CA chart (it automatically creates a postgresql database)

    helm install stable/hlf-ca -n ca --namespace blockchain -f ./helm_values/ca_values.yaml

Get pod for CA release

    CA_POD=$(kubectl get pods -n blockchain -l "app=hlf-ca,release=ca" -o jsonpath="{.items[0].metadata.name}")

Check if server is ready

    kubectl logs -n blockchain $CA_POD | grep "Listening on"

Check that we don't have a certificate

    kubectl exec -n blockchain $CA_POD -- cat /var/hyperledger/fabric-ca/msp/signcerts/cert.pem

    kubectl exec -n blockchain $CA_POD -- bash -c 'fabric-ca-client enroll -d -u http://$CA_ADMIN:$CA_PASSWORD@$SERVICE_DNS:7054'

Check that ingress works correctly

    CA_INGRESS=$(kubectl get ingress -n blockchain -l "app=hlf-ca,release=ca" -o jsonpath="{.items[0].spec.rules[0].host}")

    curl -v https://$CA_INGRESS/cainfo

    FABRIC_CA_CLIENT_HOME=./config fabric-ca-client getcacert -u https://$CA_INGRESS -M ./iTradeMSP

#### Identities

##### Organization admin

Get identity of org-admin (this should not exist at first)

    kubectl exec -n blockchain $CA_POD -- fabric-ca-client identity list --id org-admin

Register Organization admin if the previous command did not work

    kubectl exec -n blockchain $CA_POD -- fabric-ca-client register --id.name org-admin --id.secret OrgAdm1nPW --id.attrs 'admin=true:ecert'

Enroll the Organization Admin identity (typically we would use a more secure password than `OrgAdm1nPW`, etc.)

    FABRIC_CA_CLIENT_HOME=./config fabric-ca-client enroll -u https://org-admin:OrgAdm1nPW@$CA_INGRESS -M ./iTradeMSP

Copy the signcerts to admincerts

    mkdir -p ./config/iTradeMSP/admincerts

    cp ./config/iTradeMSP/signcerts/* ./config/iTradeMSP/admincerts

Create a secret to hold the admincert

    ORG_CERT=$(ls ./config/iTradeMSP/admincerts/cert.pem)

    kubectl create secret generic -n blockchain hlf--org-admincert --from-file=cert.pem=$ORG_CERT

Find the adminkey and create a secret to hold it

    ORG_KEY=$(ls ./config/iTradeMSP/keystore/*_sk)

    kubectl create secret generic -n blockchain hlf--org-adminkey --from-file=key.pem=$ORG_KEY

Create a secret to hold the CA certificate:

    CA_CERT=$(ls ./config/iTradeMSP/cacerts/*.pem)

    kubectl create secret generic -n blockchain hlf--ca-cert --from-file=cacert.pem=$CA_CERT

### Crypto material

    cd ./config

Create Genesis block and Channel

    FABRIC_CFG_PATH=$PWD configtxgen -profile OrdererGenesis -outputBlock ./genesis.block

    FABRIC_CFG_PATH=$PWD configtxgen -profile MyChannel -channelID mychannel -outputCreateChannelTx ./mychannel.tx

Save them as secrets

    kubectl create secret generic -n blockchain hlf--genesis --from-file=genesis.block

    kubectl create secret generic -n blockchain hlf--channel --from-file=mychannel.tx

    cd ..

### Kafka for Ordering service

Install Kafka chart (use special values to ensure 4 Kafka brokers and that Kafka messages don't disappear)

    helm install incubator/kafka -n kafka-hlf --namespace blockchain -f ./helm_values/kafka-hlf_values.yaml

### Fabric Orderer

For each orderer set the `NUM` environmental variable and follow the below instructions (in this example, either 1 or 2):

    export NUM=1

#### Crypto material

Register orderer with CA (typically we would use a more secure password than `ord1_pw`, etc.)

    kubectl exec -n blockchain $CA_POD -- fabric-ca-client register --id.name ord${NUM} --id.secret ord${NUM}_pw --id.type orderer

    FABRIC_CA_CLIENT_HOME=./config fabric-ca-client enroll -d -u https://ord${NUM}:ord${NUM}_pw@$CA_INGRESS -M ord${NUM}_MSP

Save the Orderer certificate in a secret

    NODE_CERT=$(ls ./config/ord${NUM}_MSP/signcerts/*.pem)

    kubectl create secret generic -n blockchain hlf--ord${NUM}-idcert --from-file=cert.pem=${NODE_CERT}

Save the Orderer private key in another secret

    NODE_KEY=$(ls ./config/ord${NUM}_MSP/keystore/*_sk)

    kubectl create secret generic -n blockchain hlf--ord${NUM}-idkey --from-file=key.pem=${NODE_KEY}

#### Helm charts

Install orderers

    helm install stable/hlf-ord -n ord${NUM} --namespace blockchain -f ./helm_values/ord${NUM}_values.yaml

Get logs from orderer to check it's actually started

    ORD_POD=$(kubectl get pods -n blockchain -l "app=hlf-ord,release=ord${NUM}" -o jsonpath="{.items[0].metadata.name}")

    kubectl logs -n blockchain $ORD_POD | grep 'completeInitialization'
    
    export NUM=2

> Repeat all above steps for Orderer 2, etc.

### Fabric Peer

For each peer set the `NUM` environmental variable and follow the below instructions (in this example, either 1 or 2):

    export NUM=1

#### Crypto material

Register orderer with CA (typically we would use a more secure password than `peer1_pw`, etc.)

    kubectl exec -n blockchain $CA_POD -- fabric-ca-client register --id.name peer${NUM} --id.secret peer${NUM}_pw --id.type peer

    FABRIC_CA_CLIENT_HOME=./config fabric-ca-client enroll -d -u https://peer${NUM}:peer${NUM}_pw@$CA_INGRESS -M peer${NUM}_MSP

Save the Orderer certificate in a secret

    NODE_CERT=$(ls ./config/peer${NUM}_MSP/signcerts/*.pem)

    kubectl create secret generic -n blockchain hlf--peer${NUM}-idcert --from-file=cert.pem=${NODE_CERT}

Save the Orderer private key in another secret

    NODE_KEY=$(ls ./config/peer${NUM}_MSP/keystore/*_sk)

    kubectl create secret generic -n blockchain hlf--peer${NUM}-idkey --from-file=key.pem=${NODE_KEY}

Install CouchDB chart

    helm install stable/hlf-couchdb -n cdb-peer${NUM} --namespace blockchain -f ./helm_values/cdb-peer${NUM}_values.yaml

Check that CouchDB is running

    CDB_POD=$(kubectl get pods -n blockchain -l "app=hlf-couchdb,release=cdb-peer${NUM}" -o jsonpath="{.items[*].metadata.name}")

    kubectl logs -n blockchain $CDB_POD | grep 'Apache CouchDB has started on'

Install Peer

    helm install stable/hlf-peer -n peer${NUM} --namespace blockchain -f ./helm_values/peer${NUM}_values.yaml

Check that Peer is running

    PEER_POD=$(kubectl get pods -n blockchain -l "app=hlf-peer,release=peer${NUM}" -o jsonpath="{.items[0].metadata.name}")

    kubectl logs -n blockchain $PEER_POD | grep 'Starting peer'
    
    export NUM=2

> Repeat all above steps for Peer 2, etc.

#### Channels

Create channel (do this only once in Peer 1)

    export NUM=1
    
    PEER_POD=$(kubectl get pods -n blockchain -l "app=hlf-peer,release=peer${NUM}" -o jsonpath="{.items[0].metadata.name}")

    kubectl exec -n blockchain $PEER_POD -- peer channel create -o ord1-hlf-ord.blockchain.svc.cluster.local:7050 -c mychannel -f /hl_config/channel/mychannel.tx

Fetch and join channel

    kubectl exec -n blockchain $PEER_POD -- peer channel fetch config /var/hyperledger/mychannel.block -c mychannel -o ord1-hlf-ord.blockchain.svc.cluster.local:7050

    kubectl exec -n blockchain $PEER_POD -- bash -c 'CORE_PEER_MSPCONFIGPATH=$ADMIN_MSP_PATH peer channel join -b /var/hyperledger/mychannel.block'
    
    export NUM=2
    
    PEER_POD=$(kubectl get pods -n blockchain -l "app=hlf-peer,release=peer${NUM}" -o jsonpath="{.items[0].metadata.name}")

> Repeat above 2 commands (`fetch` & `join`) for Peer 2, etc.

Check which channels the peer has joined:

    kubectl exec $PEER_POD -n blockchain -- peer channel list

## Deleting

Delete helm deployments

    helm delete --purge ca-pg ca kafka-hlf ord1 ord2 cdb-peer1 peer1 cdb-peer2 peer2

Delete stateful sets (in case Helm does not fully delete them)

    kubectl delete statefulset -n blockchain kafka-log kafka-hlf-zookeeper kafka-hlf

Delete Persistent Volume Claims

    kubectl delete pvc -n blockchain ca-pg-postgresql data-kafka-hlf-zookeeper-0 data-kafka-hlf-zookeeper-1 data-kafka-hlf-zookeeper-2 datadir-kafka-hlf-0 datadir-kafka-hlf-1 datadir-kafka-hlf-2 datadir-kafka-hlf-3

Delete secrets on K8S

    kubectl delete secret -n blockchain hlf--org-admincert  hlf--org-adminkey hlf--channel hlf--genesis

Delete crypto material

    rm -rf ./config/*MSP ./config/genesis.block ./config/mychannel.tx

## Extra resources

### Repositories

Our charts can be found at the official Helm Charts repository:

https://github.com/helm/charts

And also on our own open-source repository:

https://github.com/aidtechnology/at-charts

We also have a repository hosting the Fabric CA client Homebrew installer (for OS X):

https://github.com/aidtechnology/homebrew-fabric-ca

### Courses

*Blockchain for Business - An Introduction to Hyperledger Technologies*, where we have contributed the Hyperledger Composer chapter:

https://www.edx.org/course/blockchain-business-introduction-linuxfoundationx-lfs171x-0

*Blockchain for Blockchain Applications* on Packt and Udemy:

https://www.packtpub.com/application-development/hyperledger-blockchain-applications-video

https://www.udemy.com/hyperledger-for-blockchain-applications/

### FAQ

> In progress
