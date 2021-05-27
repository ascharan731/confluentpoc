===============================
This enviroment set up instructions are taken from  `Confluent<https://github.com/confluentinc/confluent-kubernetes-examples/tree/master/security/secure-authn-encrypt-deploy>`__.
===============================


================================
Deploy Secure Confluent Platform
================================

In this workflow scenario, you'll set up secure Confluent Platform clusters with
mTLS authentication, no authorization, and inter-component TLS.


To complete this scenario, you'll follow these steps:

#. Set the current working  directory.

#. Deploy Confluent For Kubernetes.

#. Deploy configuration secrets.

#. Deploy Confluent Platform.

#. Tear down Confluent Platform.

==================================
Set the current workiing directory
==================================

Clone the git repo and set the current working directory 

::
  git clone <git url>
   
  export WORKING_DIR=<working directory>/secure-authn-encrypt-deploy
  
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

============================
Deploy configuration secrets
============================

   
Provide a Root Certificate Authority
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

#. Generate a CA pair to use in this scenario: 

   ::

     openssl genrsa -out $WORKING_DIR/ca-key.pem 2048
    
   ::

     openssl req -new -key $WORKING_DIR/ca-key.pem -x509 \
       -days 1000 \
       -out $WORKING_DIR/ca.pem \
       -subj "/C=US/ST=CA/L=MountainView/O=Confluent/OU=Operator/CN=TestCA"

#. Create a Kuebernetes secret for inter-component TLS:

   ::

     kubectl create secret tls ca-pair-sslcerts \
       --cert=$WORKING_DIR/ca.pem \
       --key=$WORKING_DIR/ca-key.pem
  
Provide authentication credentials by creating secret object for zookeeper, kafka, and Control Center.
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

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

     kubectl apply -f $WORKING_DIR/confluent-platform-secure.yaml

#. Check that all Confluent Platform resources are deployed:

   ::
   
     kubectl get confluent

#. Get the status of any component. For example, to check Control Center:

   ::
   
     kubectl describe controlcenter

Validate in Control Center
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

  kubectl delete -f $WORKING_DIR/confluent-platform-secure.yaml

::

  kubectl delete secret kafka-client-config-secure

::

  kubectl delete secret credential

::

  kubectl delete secret ca-pair-sslcerts

::

  helm delete operator
  
