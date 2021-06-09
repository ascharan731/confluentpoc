===============================
This enviroment set up instructions are taken from  `Confluent <https://github.com/confluentinc/confluent-kubernetes-examples/tree/master/security/secure-authn-encrypt-deploy>`_ .
===============================


================================
Deploy Secure Confluent Platform
================================

In this workflow scenario, you'll set up secure Confluent Platform clusters with
mTLS authentication.


To complete this scenario, you'll follow these steps:

#. Set the current working  directory.

#. Deploy Confluent For Kubernetes.

#. Generate certificates 

#. Deploy configuration secrets.

#. Deploy Confluent Platform.

#. Tear down Confluent Platform.

==================================
Set the current workiing directory
==================================

Clone the git repo and set the current working directory 

::

  git clone <git url>
   
  export WORKING_DIR=<working directory>/confluentpoc
  
===============================
Deploy Confluent for Kubernetes
===============================
#. Create confluent namespace

   ::
   
     kubectl create namespace confluent

#. Set up the Helm Chart:

   ::

     helm repo add confluentinc https://packages.confluent.io/helm


#. Install Confluent For Kubernetes using Helm:

   ::

     helm upgrade --install operator confluentinc/confluent-for-kubernetes
  
#. Check that the Confluent For Kubernetes pod comes up and is running:

   ::
     
     kubectl get pods
.

============================
Generate certificates
============================

   
Creating root certificates
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

#. create CA key pair to use in this scenario: 

   ::
     
     mkdir $WORKING_DIR/certs/generated
     
     cfssl gencert -initca $WORKING_DIR/certs/ca-csr.json | cfssljson -bare $WORKING_DIR/certs/generated/ca -

#. Validate Certificate Authority

   :: 
   
     openssl x509 -in $WORKING_DIR/certs/generated/ca.pem -text -noout
    
#. Make up server-domain.json files, below is for zookeeper, likewise, create for other confluent components. 

   ::   
   
     cat $WORKING_DIR/certs/zookeeper-domain.json
          
           {
              "CN": "zookeeper",
              "hosts": [
                  "*.confluent.svc.cluster.local",
                  "*.zookeeper.confluent.svc.cluster.local",
                  "*.kafka.confluent.svc.cluster.local"
              ],
              "key": {
                 "algo": "rsa",
                  "size": 2048
              },
             "names": [
               {
                 "C": "Universe",
                 "ST": "Pangea",
                 "L": "Earth"
               }
              ]
           }

#. Create server certificates for each component as  below 

   ::
   
     cfssl gencert -ca=$WORKING_DIR/certs/generated/ca.pem \
     -ca-key=$WORKING_DIR/certs/generated/ca-key.pem \
     -config=$WORKING_DIR/certs/ca-config.json \
     -profile=server $WORKING_DIR/certs/zookeeper-domain.json | cfssljson -bare $WORKING_DIR/certs/generated/zookeeper

#. Validate server certificate 

   ::
   
     openssl x509 -in $WORKING_DIR/certs/generated/zookeeper.pem -text -noout
     
============================
Deploy configuration secrets
============================

#. Create a Kubernetes secrets for zookeeper, likewise, create for other confluent components:

   ::
   
     kubectl create secret generic tls-zookeeper \
     --from-file=fullchain.pem=$WORKING_DIR/certs/generated/zookeeper.pem \
     --from-file=cacerts.pem=$WORKING_DIR/certs/generated/ca.pem \
     --from-file=privkey.pem=$WORKING_DIR/certs/generated/zookeeper-key.pem
  

Provide authentication credentials 
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

::

  kubectl create secret generic credential \
  --from-file=plain-users.json=$WORKING_DIR/creds-kafka-sasl-users.json \
  --from-file=digest-users.json=$WORKING_DIR/creds-zookeeper-sasl-digest-users.json \
  --from-file=digest.txt=$WORKING_DIR/creds-kafka-zookeeper-credentials.txt \
  --from-file=plain.txt=$WORKING_DIR/creds-client-kafka-sasl-user.txt \
  --from-file=basic.txt=$WORKING_DIR/creds-control-center-users.txt


=========================
Deploy Confluent Platform
=========================

#. Deploy Confluent Platform with the above configuration:

   ::

     kubectl apply -f $WORKING_DIR/confluent-platform-production-mtls.yaml

#. Check that all Confluent Platform resources are deployed:

   ::
   
     kubectl get confluent

#. Get the status of any component. For example, to check Control Center:

   ::
   
     kubectl describe controlcenter

Access control center
^^^^^^^^^^^^^^^^^^^^^^^^^^


#. Set up port forwarding to Control Center web UI from local machine:

   ::

     kubectl port-forward controlcenter-0 8021:8021

#. Browse to Control Center and log in as the ``admin`` user with the ``Developer1`` password:

   ::
   
     https://localhost:8021


=========
Tear down
=========

::

  kubectl delete -f $WORKING_DIR/confluent-platform-production-mtls.yaml


::

  kubectl delete secret credential


::

  helm delete operator
  
