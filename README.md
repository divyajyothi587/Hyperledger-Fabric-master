# Hyperledger Fabric on Kubernetes

Hyperledger Fabric is a platform for distributed ledger solutions underpinned by a modular architecture delivering high degrees of confidentiality, resiliency, flexibility, and scalability. It is designed to support pluggable implementations of different components and accommodate the complexity and intricacies that exist across the economic ecosystem.


## What is a Blockchain?

Its better you can read the complete [docs](https://hyperledger-fabric.readthedocs.io/en/release-1.4/blockchain.html) before jimp into the deployment.

### A Distributed Ledger

At the heart of a blockchain network is a distributed ledger that records all the transactions that take place on the network.

A blockchain ledger is often described as decentralized because it is replicated across many network participants, each of whom collaborate in its maintenance. We’ll see that decentralization and collaboration are powerful attributes that mirror the way businesses exchange goods and services in the real world.


### Smart Contracts

To support the consistent update of information — and to enable a whole host of ledger functions (transacting, querying, etc) — a blockchain network uses smart contracts to provide controlled access to the ledger.

Smart contracts are not only a key mechanism for encapsulating information and keeping it simple across the network, they can also be written to allow participants to execute certain aspects of transactions automatically.


### Consensus

The process of keeping the ledger transactions synchronized across the network — to ensure that ledgers update only when transactions are approved by the appropriate participants, and that when ledgers do update, they update with the same transactions in the same order — is called consensus.


**Note :** For Hyperledger Fabric Deployment, please run the commands [./hlf.sh -o options](https://raw.githubusercontent.com/akhilrajmailbox/Hyperledger-Fabric/master/MyProd/hlf.sh) within [MyProd folder](https://github.com/akhilrajmailbox/Hyperledger-Fabric/tree/master/MyProd)


## Hardcoded Environment Variables in the shell script `hlf.sh`

```
export CUSTOM_PEERS="1 2 3 4"
export ANCHOR_PEERS="1 2"
export CUSTOM_ORG_NUMS="1 2"
```

These hardcoded values can be changed according to your cluster size, in this example we configured a cluster of 4 peers per org and out of 4 peers 2 peers are configured as anchor peer



## Production Deployment

[hyperledger fabric docs](https://hyperledger-fabric.readthedocs.io)

[hlf on k8s](#https://github.com/IBM/blockchain-network-on-kubernetes.git)

## Prerequisite

* go version 1.13.4
* envsubt
* GOHOME in your PATH Environment variable # export PATH=/Users/akhil/go/bin:$PATH
* [fabric-tools](https://github.com/hyperledger/homebrew-fabric/tree/master/Formula) for "configtxgen"
* kubectl
* [helm](https://helm.sh/docs/intro/install/)


### Install fabric-ca-client on your Mac / Linux system

```
go version
go get -u github.com/hyperledger/fabric-ca/cmd/...
```

### Install [Fabric-tools](https://github.com/hyperledger/homebrew-fabric/tree/master/Formula) on your Mac

```
brew tap aidtechnology/homebrew-fabric
xcode-select --install
brew install aidtechnology/fabric/fabric-tools@1.3.0

which cryptogen
which configtxgen
which configtxlator
```


### Create your K8s Cluster on AWS / Azure (below example only for AWS)

[install / upgrade eksctl](https://docs.aws.amazon.com/eks/latest/userguide/eksctl.html)

```
eksctl create cluster \
    --name HLF-Cluster \
    --version 1.14 \
    --zones us-east-1a,us-east-1b,us-east-1c \
    --region us-east-1 \
    --nodegroup-name HLF-Cluster-Nodeworkers \
    --node-type t3a.2xlarge \
    --nodes 5 \
    --nodes-min 5 \
    --nodes-max 5 \
    --managed
```

**create eks cluster with exsisting VPC**

choose the following option according to your vpc network configuration

* --vpc-private-subnets
* --vpc-public-subnets


```
eksctl create cluster \
    --name HLF-Cluster \
    --version 1.14 \
    --region us-east-1 \
    --vpc-public-subnets subnet-f8b2d09f,subnet-3acaa314,subnet-41d4e60b \
    --nodegroup-name HLF-Cluster-Nodeworkers \
    --node-type t3a.2xlarge \
    --nodes 5 \
    --nodes-min 5 \
    --nodes-max 5 \
    --managed
```

[Configure Cert-manager with Let's Encrypt](https://cert-manager.io/docs/tutorials/acme/ingress/)

### [Debug Cert-manager](https://github.com/jetstack/cert-manager/issues/2020) ; You can actually find error messages in each of these, like so:

```
kubectl get certificaterequest
kubectl describe certificaterequest X
kubectl get order
kubectl describe order X
kubectl get challenge
kubectl describe challenge X
```

**WARNING : Change the following files**

* configtx.yml
* ca.yaml
    * hosts
    * externalDatabase
    * config
* Chaincode-Jobs
    * chaincode-ConfigMap.yaml
        * CHANNEL_NAME
        * CHAINCODE_NAME
        * LANGUAGE
    * chaincode_install.yaml
        * update the bash scripts according to your chaincode language
* efs-provisioner-deployment.yml
    * file.system.id
    * aws.region
    * server


## K8s Deployment

**WARNING : Configure your `configtx.yml` according to your needs, once you deploy the network then there is no option to reconfigure it without a downtime**

[configtx-example](https://github.com/hyperledger/fabric-sdk-go/blob/master/test/fixtures/fabric/v1.3/config/configtx.yaml)


### The existing fabric network has the following components.

* 5 orderer nodes
* 4 peer nodes per organisations (8 nodes total) --->  2 peer out of 4 per organisation configured as AnchorPeers (peer1 and peer2) 
* 2 organisations (Org1 and Org2)
* 3 channels
    * org1channel       -->     Channel for Org1 (Intra)
    * org2channel       -->     channel for Org2 (Intra)
    * twoorgschannel    -->     Channel for both Org1 and Org2 (Inter)
* External HA Kafka Cluster
* External MySQL Database Server for Fabric CA
* One sub domain for Fabric CA


### This repo have the following custom configuration in the file : `configtx.yml` for the fabric network


* Organizations
    * OrdererOrg
        * Policies : OrdererOrgPolicies
    * Org1
        * Policies : Org1Policies
    * Org2
        * Policies : Org1Policies

* Capabilities
    * Channel : V1_1
    * Orderer : V1_1
    * Application : V1_2

* Application : ApplicationDefaults
    * Policies : ANY Readers/ANY Writers/MAJORITY Admins
    * Capabilities : ApplicationCapabilities

* Orderer : OrdererDefaults
    * Addresses : (3 nodes)
    * Kafka : (3 Brokers)
    * Policies : ANY Readers/ANY Writers/MAJORITY Admins

* Channel : ChannelDefaults
    * Policies : ANY Readers/ANY Writers/MAJORITY Admins
    * Capabilities : ChannelCapabilities

* Profiles
    * OrdererGenesis : ChannelDefaults
        * Orderer
        * Consortiums

            * Org1Consortium
                * Organizations
                    * Org1
                        * Policies : Org1Policies

            * Org2Consortium
                * Organizations
                    * Org2
                        * Policies : Org2Policies

            * TwoOrgsConsortium
                * Organizations
                    * Org1
                        * Policies : Org1Policies
                    * Org2
                        * Policies : Org2Policies

    * Org1Channel
        * Consortium : Org1Consortium
        * Organizations
            * Org1

    * Org2Channel
        * Consortium : Org2Consortium
        * Organizations
            * Org2

    * TwoOrgsChannel
        * Consortium : TwoOrgsConsortium
        * Organizations
            * Org1
            * Org2



### Main modes of operation

#### Initialisation for the HLF Cluster, It will create the following resources in the kubernetes cluster:

* fast storageclass
* nginx ingress
* namespaces (cas, orderers and peers)

```
./hlf.sh -o initial
```

#### CA Mager Configuration

Install and configure Cert Manager to get the SSL certificates by using Let's Encrypt Account

*This function will create the following resources in the kubernetes cluster*

* CRDs (CustomResourceDefinition)
* certManagerCI_staging
* certManagerCI_production

```
./hlf.sh -o cert-manager
```

#### Fabric CA

Install and configure Fabric CA on K8s namespace cas.

**Note : Before running this command, please customise helm_values/ca.yaml with your details ; check for TODO hashtag in that file....**

```
./hlf.sh -o fabric-ca
```
**check your fabric-ca deployment**

```
curl https://CA_INGRESS_DOMAIN/cainfo
```

#### Org Orderer Organisation Identities

Configure the Orderer Admin in your network

*This function will create the following resources in the kubernetes cluster*

* OrdererMSP
* identity creation : ord-admin in CA Server
* K8s secrets in `orderers` namespace
    * hlf--ord-admincert
    * hlf--ord-adminkey
    * hlf--ord-ca-cert

```
./hlf.sh -o org-orderer-admin
```

#### Org Peer Organisation Identities

Configure the Peer Admin in your network for each Organisation; `example : "N" peer admin configuration for "N" organisation`

*This function will create the following resources in the kubernetes cluster*

* Org1MSP
* Org2MSP
* Org`N`MSP
* identity creation : peer-org1-admin / peer-org2-admin / peer-org`N`-admin etc... in CA Server
* K8s secrets in `peers` namespace
    * hlf--peer-org1-admincert / hlf--peer-org2-admincert / hlf--peer-org`N`-admincert etc...
    * hlf--peer-org1-adminkey / hlf--peer-org2-adminkey / hlf--peer-org`N`-adminkey etc...
    * hlf--peer-org1-ca-cert / hlf--peer-org2-ca-cert / hlf--peer-org`N`-ca-cert etc...

```
./hlf.sh -o org-peer-admin
```

#### Create Genesis block

Create the Genesis Block with your configuration with `configtx.yml`

*This function will create the following resources in the kubernetes cluster*

* genesis.block with channelID `systemchannel`
* secret : `hlf--genesis` in `orderers` namespace >> Optional

```
./hlf.sh -o genesis-block
```

#### Create the Channel Block

Create the Channel with your configuration with `configtx.yml`

**IMPORTANT : Don't Use the channel name : `systemchannel` because it is reserverd for system channel for our network ;  channel name must be in lowercase / numbers**

*Channel Option also need to choose while Creating the Channel Block*

```
Org1Channel     :   Channel for Org1; only Intra communication is possible between Org1 peers...!
Org2Channel     :   Channel for Org2; only Intra communication is possible between Org2 peers...!
TwoOrgsChannel  :   Channel for both Org1 and Org2, Inter Communication between Organisations...!
```

*This function will create the following resources in the kubernetes cluster*

* CHANNEL_NAME.tx
* secret : `hlf--channel` in `peers` namespace >> Optional


```
./hlf.sh -o channel-block
```

#### Orderer Node Deployment

Create the Orderers certs and configure it in the K8s secrets, Deploying the Orderers nodes on namespace orderers. Create "N" number of orderers which mentioned in `configtx.yaml`

*This function will create the following resources in the kubernetes cluster*

* ord1_MSP
* ord2_MSP
* ord`N`_MSP
* identity creation : ord1 / ord2 / ord`N` etc... in CA Server
* K8s secrets in `orderers` namespace
    * hlf--ord1-idcert / hlf--ord2-idcert / hlf--ord`N`-idcert etc...
    * hlf--ord1-idkey / hlf--ord2-idkey / hlf--ord1-`N`dkey etc...

```
./hlf.sh -o orderer-create
```

#### Peer Node Deployment

Create the Orderers certs and configure it in the K8s secrets, Deploying the Peers nodes on namespace peers. Create "N" Number of peers for "N" Orderers == "N*N".

*This function will create the following resources in the kubernetes cluster*

* peer1-org1_MSP / peer1-org2_MSP / peer1-org`N`_MSP etc...
* peer2-org1_MSP / peer2-org2_MSP / peer2-org`N`_MSP etc...
* peer`N`-org1_MSP / peer`N`-org2_MSP / peer`N`-org`N`_MSP etc...
* identity creation : peer1-org1 / peer1-org2 / peer2-org1 / peer2-org2 / peer`N`-org`N` etc... in CA Server
* K8s secrets in `peers` namespace
    * hlf--peer1-org1-idcert / hlf--peer1-org2-idcert / hlf--peer2-org1-idcert / hlf--peer2-org2-idcert / hlf--peer`N`-org`N`-idcert
    * hlf--peer1-org1-idkey / hlf--peer1-org2-idkey / hlf--peer2-org1-idkey / hlf--peer2-org2-idkey / hlf--peer`N`-org`N`-idkey

```
./hlf.sh -o peer-create
```

#### Creating the channel on peer

One time configuraiton on first peer (peer-org1-1 / peer-org2-1) ; In next step we will fetch and join it from all peers

**Note : You must create the channel with excatly same name what you are giving as a input in this command with the command : `channel-block` in your local before execute this function**
**IMPORTANT : Don't Use the channel name : `systemchannel` because it is reserverd for system channel for our network ;  channel name must be in lowercase / numbers**

```
./hlf.sh -o channel-create
```

#### Join to the channel from required peers

Once you created the channel in the fabric network with the above command, then you have to join the needed peers from each organisation to communicate with eachother.

```
./hlf.sh -o channel-join
```

#### List the channels in a peer

List all channels which a particular peer has joined.

```
./hlf.sh -o channel-ls
```

## Configure `tlscerts` configuration

[tlscerts](#https://github.com/Blockdaemon/fabric-ca/blob/gerrit-pr-29430/docs/source/operations_guide.rst)


## First Time Deployment :

```
./hlf.sh -o initial
./hlf.sh -o cert-manager
./hlf.sh -o fabric-ca
./hlf.sh -o org-orderer-admin
./hlf.sh -o org-peer-admin ---> ("N" peer admin configuration for "N" organisation)
./hlf.sh -o genesis-block
./hlf.sh -o channel-block
./hlf.sh -o orderer-create (Create "N" number of orderers which mentioned in "configtx.yaml")
./hlf.sh -o peer-create ---> (Create "N" Number of peers for "N" Orderers == "N*N")
./hlf.sh -o channel-create (One time configuration, run this only on one peer per Organisation [ peer-org1-1 / peer-org2-1 ])
./hlf.sh -o channel-join ---> (Run on "N" Peers on all Organisation)
./hlf.sh -o anchor-config
./hlf.sh -o cc-deploy
./hlf.sh -o channel-ls
./hlf.sh -o cc-install-ls 
./hlf.sh -o cc-instantiate-ls
```


### Kafka Topics Management :

If you are using Managed Kafka Services as Message brocker, then after testing or if you want to recreate the channel with same channel name which you used before; then try to delete the topics from kafka and then configure the fabric network.

**You must need to configure Kafka client in your system in order to run the scripts, Put this script under kafka bin folder**

```
sudo mkdir /opt/kafka && cd /opt/kafka
sudo curl "https://www.apache.org/dist/kafka/2.1.1/kafka_2.11-2.1.1.tgz" -o ./kafka.tgz
sudo tar -xzvf kafka.tgz --strip 1
sudo rm -rf kafka.tgz
sudo curl https://raw.githubusercontent.com/akhilrajmailbox/Hyperledger-Fabric-K8s/master/MyProd/extra/kafka-mytopic.sh -o /opt/kafka/bin/kafka-mytopic.sh
sudo chmod a+x /opt/kafka/bin/kafka-mytopic.sh
cd /opt/kafka/bin/  ## update your zookeeper ip address in the script : kafka-mytopic.sh
./kafka-mytopic.sh -o [OPTION...]
```




### helm charts repository creation steps.

How It Works

Create github repository and create docs folder in root directory.

The docs folder contains index.html file

set up GitHub Pages to point to the docs folder.

From there, I can create and publish docs like this:

```
$ helm create hlf-ca
$ cd hlf-ca ; helm dependency update ; cd -
$ helm package hlf-ca
$ mv hlf-ca-0.1.0.tgz docs
$ helm repo index docs --url https://akhilrajmailbox.github.io/Hyperledger-Fabric/docs
$ git add .
$ git commit -m "updated" -av
$ git push origin master
```

add helm repo to your system and install.

```
$ helm repo add ar-repo https://akhilrajmailbox.github.io/Hyperledger-Fabric/docs
```