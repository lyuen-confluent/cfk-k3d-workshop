Production recommended secure setup (K3d)
=========================================

Confluent recommends these security mechanisms for a production deployment:

- Enable Kafka client authentication. Choose one of:

  - SASL/Plain or mTLS

  - For SASL/Plain, the identity can come from LDAP server

- Enable Confluent Role Based Access Control for authorization, with user/group identity coming from LDAP server

- Enable TLS for network encryption - both internal (between CP components) and external (Clients to CP components)

In this deployment scenario, we will set this up, choosing SASL/Plain for authentication, using user provided custom certificates.

==================================
Set the current tutorial directory
==================================

Set the tutorial directory for this tutorial under the directory you downloaded
the tutorial files:

::
   
  export TUTORIAL_HOME=/home/training/confluent-operator/security/production-secure-deploy
  
===============================
Deploy Confluent for Kubernetes
===============================

#. Set up the Helm Chart:

   ::

     helm repo add confluentinc https://packages.confluent.io/helm

Note that it is assumed that your Kubernetes cluster has a ``confluent`` namespace available, otherwise you can create it by running ``kubectl create namespace confluent``. 


#. Install Confluent For Kubernetes using Helm:

   ::

     helm upgrade --install operator confluentinc/confluent-for-kubernetes --namespace confluent
  
#. Check that the Confluent For Kubernetes pod comes up and is running:

   ::
     
     kubectl get pods --namespace confluent

===============
Deploy OpenLDAP
===============

This repo includes a Helm chart for `OpenLdap
<https://github.com/osixia/docker-openldap>`__. The chart ``values.yaml``
includes the set of principal definitions that Confluent Platform needs for
RBAC.

#. Deploy OpenLdap

   ::

     helm upgrade --install -f $TUTORIAL_HOME/../../assets/openldap/ldaps-rbac.yaml test-ldap $TUTORIAL_HOME/../../assets/openldap --namespace confluent

#. Validate that OpenLDAP is running:  
   
   ::

     kubectl get pods --namespace confluent

#. Log in to the LDAP pod:

   ::

     kubectl --namespace confluent exec -it ldap-0 -- bash

#. Run the LDAP search command:

   ::

     ldapsearch -LLL -x -H ldap://ldap.confluent.svc.cluster.local:389 -b 'dc=test,dc=com' -D "cn=mds,dc=test,dc=com" -w 'Developer!'

#. Exit out of the LDAP pod:

   ::
   
     exit 
     
============================
Deploy configuration secrets
============================

You'll use Kubernetes secrets to provide credential configurations.

With Kubernetes secrets, credential management (defining, configuring, updating)
can be done outside of the Confluent For Kubernetes. You define the configuration
secret, and then tell Confluent For Kubernetes where to find the configuration.
   
To support the above deployment scenario, you need to provide the following
credentials:

* Component TLS Certificates

* Authentication credentials for Zookeeper, Kafka, Control Center, remaining CP components

* RBAC principal credentials
  
Create your own certificates
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

When testing, it's often helpful to generate your own certificates to validate the architecture and deployment.

You'll want both these to be represented in the certificate SAN:

- external domain names
- internal Kubernetes domain names

The internal Kubernetes domain name depends on the namespace you deploy to. If you deploy to `confluent` namespace, 
then the internal domain names will be: 

- \*.kafka.confluent.svc.cluster.local
- \*.zookeeper.confluent.svc.cluster.local
- \*.confluent.svc.cluster.local

#. Update $TUTORIAL_HOME/../../assets/certs/server-domain.json to include external domain name "confluent.edu". The file should contain the following:

   ::

    {
      "CN": "*.confluent.edu",
      "hosts": [
      "*.confluent.edu",
      "*.confluent.svc.cluster.local",
      "*.zookeeper.confluent.svc.cluster.local",
      "*.kafka.confluent.svc.cluster.local",
      "*.ldap.confluent.svc.cluster.local",
      "ldap.confluent.svc.cluster.local",
      "*.my.domain",
      "*.destination.svc.cluster.local",
      "*.zookeeper.destination.svc.cluster.local",
      "*.kafka.destination.svc.cluster.local",
      "*.replicator.destination.svc.cluster.local"
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

#. Create Certificate Authority
  
   ::

    mkdir $TUTORIAL_HOME/../../assets/certs/generated && cfssl gencert -initca $TUTORIAL_HOME/../../assets/certs/ca-csr.json | cfssljson -bare $TUTORIAL_HOME/../../assets/certs/generated/ca -

#. Validate Certificate Authority

   ::
  
    openssl x509 -in $TUTORIAL_HOME/../../assets/certs/generated/ca.pem -text -noout

#. Create server certificates with the appropriate SANs (SANs listed in server-domain.json)

   ::

    cfssl gencert -ca=$TUTORIAL_HOME/../../assets/certs/generated/ca.pem \
    -ca-key=$TUTORIAL_HOME/../../assets/certs/generated/ca-key.pem \
    -config=$TUTORIAL_HOME/../../assets/certs/ca-config.json \
    -profile=server $TUTORIAL_HOME/../../assets/certs/server-domain.json | cfssljson -bare $TUTORIAL_HOME/../../assets/certs/generated/server

#. Validate server certificate and SANs

   ::
    
    openssl x509 -in $TUTORIAL_HOME/../../assets/certs/generated/server.pem -text -noout



Provide component TLS certificates
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

::
   
    kubectl create secret generic tls-group1 \
      --from-file=fullchain.pem=$TUTORIAL_HOME/../../assets/certs/generated/server.pem \
      --from-file=cacerts.pem=$TUTORIAL_HOME/../../assets/certs/generated/ca.pem \
      --from-file=privkey.pem=$TUTORIAL_HOME/../../assets/certs/generated/server-key.pem \
      --namespace confluent


Provide authentication credentials
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

#. Create a Kubernetes secret object for Zookeeper, Kafka, and Control Center.

   This secret object contains file based properties. These files are in the
   format that each respective Confluent component requires for authentication
   credentials.

   ::
   
     kubectl create secret generic credential \
       --from-file=plain-users.json=$TUTORIAL_HOME/creds-kafka-sasl-users.json \
       --from-file=digest-users.json=$TUTORIAL_HOME/creds-zookeeper-sasl-digest-users.json \
       --from-file=digest.txt=$TUTORIAL_HOME/creds-kafka-zookeeper-credentials.txt \
       --from-file=plain.txt=$TUTORIAL_HOME/creds-client-kafka-sasl-user.txt \
       --from-file=basic.txt=$TUTORIAL_HOME/creds-control-center-users.txt \
       --from-file=ldap.txt=$TUTORIAL_HOME/ldap.txt \
       --namespace confluent

   In this tutorial, we use one credential for authenticating all client and
   server communication to Kafka brokers. In production scenarios, you'll want
   to specify different credentials for each of them.

Provide RBAC principal credentials
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

#. Create a Kubernetes secret object for MDS:

   ::
   
     kubectl create secret generic mds-token \
       --from-file=mdsPublicKey.pem=$TUTORIAL_HOME/../../assets/certs/mds-publickey.txt \
       --from-file=mdsTokenKeyPair.pem=$TUTORIAL_HOME/../../assets/certs/mds-tokenkeypair.txt \
       --namespace confluent
   
   ::
   
     # Kafka RBAC credential
     kubectl create secret generic mds-client \
       --from-file=bearer.txt=$TUTORIAL_HOME/bearer.txt \
       --namespace confluent
     # Control Center RBAC credential
     kubectl create secret generic c3-mds-client \
       --from-file=bearer.txt=$TUTORIAL_HOME/c3-mds-client.txt \
       --namespace confluent
     # Connect RBAC credential
     kubectl create secret generic connect-mds-client \
       --from-file=bearer.txt=$TUTORIAL_HOME/connect-mds-client.txt \
       --namespace confluent
     # Schema Registry RBAC credential
     kubectl create secret generic sr-mds-client \
       --from-file=bearer.txt=$TUTORIAL_HOME/sr-mds-client.txt \
       --namespace confluent
     # ksqlDB RBAC credential
     kubectl create secret generic ksqldb-mds-client \
       --from-file=bearer.txt=$TUTORIAL_HOME/ksqldb-mds-client.txt \
       --namespace confluent
     # Kafka Rest Proxy RBAC credential
     kubectl create secret generic krp-mds-client \
       --from-file=bearer.txt=$TUTORIAL_HOME/krp-mds-client.txt \
       --namespace confluent
     # Kafka REST credential
     kubectl create secret generic rest-credential \
       --from-file=bearer.txt=$TUTORIAL_HOME/bearer.txt \
       --from-file=basic.txt=$TUTORIAL_HOME/bearer.txt \
       --namespace confluent

============================
Configure Confluent Platform
============================

You install Confluent Platform components as custom resources (CRs). 

You can configure all Confluent Platform components as custom resources. In this
tutorial, you will configure all components in a single file and deploy all
components with one ``kubectl apply`` command.

The CR configuration file contains a custom resource specification for each
Confluent Platform component, including replicas, image to use, resource
allocations.

Edit the Confluent Platform CR file: ``$TUTORIAL_HOME/confluent-platform-production.yaml``

you'll set up Confluent Platform component clusters
with static host-based routing to enable external clients to access
Kafka.

The Kafka external listener section of the file should be modified as follow for external access:

:: 

  spec:
    listeners:
      external:
        externalAccess:
          type: staticForHostBasedRouting
          staticForHostBasedRouting:
            domain: confluent.edu
            port: 8443
        tls:
          enabled: true

Remove the "externalAccess" section of other components

Kafka is configured with 3 replicas in this tutorial. So, the access endpoints
of Kafka will be:

* kafka.$DOMAIN for the bootstrap server
* b0.$DOMAIN for the broker #1
* b1.$DOMAIN for the broker #2
* b2.$DOMAIN for the broker #3

The access endpoint of each Confluent Platform component will be: 

::

  <Component CR name>.$DOMAIN

For example, in a brower, you will access Control Center at:

::

  https://controlcenter.$DOMAIN:8443

=========================
Deploy Confluent Platform
=========================

#. Deploy Confluent Platform:

   ::

     kubectl apply -f $TUTORIAL_HOME/confluent-platform-production.yaml --namespace confluent

#. Check that all Confluent Platform resources are deployed:

   ::
   
     kubectl get pods --namespace confluent

   If any component does not deploy, it could be due to missing configuration information in secrets.
   The Kubernetes events will tell you if there are any issues with secrets. For example:

   ::

     kubectl get events --namespace confluent
     Warning  KeyInSecretRefIssue  kafka/kafka  required key [ldap.txt] missing in secretRef [credential] for auth type [ldap_simple]

#. The default required RoleBindings for each Confluent component are created
   automatically, and maintained as `confluentrolebinding` custom resources.

   ::

     kubectl get confluentrolebinding --namespace confluent

If you'd like to see how the RoleBindings custom resources are structured, so that
you can create your own RoleBindings, take a look at the custom resources in this 
directory: $TUTORIAL_HOME/internal-rolebindings

=========================
Create bootstrap services
=========================

Kafka bootstrap
^^^^^^^^^^^^^^^

When using staticForHostBasedRouting as externalAccess type, the bootstrap
endpoint is not configured to access Kafka. 

If you want to have a bootstrap endpoint to access Kafka instead of using each
broker's endpoint, you need to provide the bootstrap endpoint, create a
DNS record pointing to Ingress controller load balancer's external IP, and
define the ingress rule for it.

Create the Kafka bootstrap service to access Kafka:

::

  kubectl apply -f /home/training/confluent-operator/networking/external-access-static-host-based/kafka-bootstrap-service.yaml

Other component bootstrap
^^^^^^^^^^^^^^^^^^^^^^^^^

::

  kubectl apply -f /home/training/confluent-operator/networking/external-access-static-host-based/connect-bootstrap-service.yaml
  kubectl apply -f /home/training/confluent-operator/networking/external-access-static-host-based/ksqldb-bootstrap-service.yaml
  kubectl apply -f /home/training/confluent-operator/security/internal_external-tls_mtls_confluent-rbac/mds-bootstrap-service.yaml

=====================================
Deploy Ingress Controller and Ingress
=====================================

In many load balancer use cases, the decryption happens at the load balancer, and then unencrypted data is passed along to the endpoint.
This is known as SSL termination.
With the Kafka protocol, however, the broker expects to perform the SSL handshake directly with the client.
To achieve this, SSL passthrough is required.
SSL passthrough is the action of passing data through a load balancer to a server without decrypting it. 
Therefore, whatever Ingress controller you choose must support SSL passthrough to access Kafka using static host-based routing.

In this lab excercise, the k3d cluster already has an ingress controller configured.

       
Create Ingress Resource
^^^^^^^^^^^^^^^^^^^^^^^

Create an Ingress resource that includes a collection of rules that the Ingress
controller uses to route the inbound traffic to Kafka.

::

  kubectl apply -f https://raw.githubusercontent.com/lyuen-confluent/cfk-k3d-workshop/main/ingress.yaml

===============
Add DNS records
===============

In production environment, you need to create DNS records for Kafka brokers using the ingress controller load balancer externalIP.
In our lab envrionment, the EXTERNAL-IP is 127.0.1.1, i.e. localhost

Your local host table (/etc/hosts) already have an entry for each components and brokers listed below: 
   
====================== ===============================================================
DNS name               IP address
====================== ===============================================================
kafka.$DOMAIN          The ``EXTERNAL-IP`` value of the ingress load balancer service
b0.$DOMAIN             The ``EXTERNAL-IP`` value of the ingress load balancer service
b1.$DOMAIN             The ``EXTERNAL-IP`` value of the ingress load balancer service
b2.$DOMAIN             The ``EXTERNAL-IP`` value of the ingress load balancer service
connect.$DOMAIN        The ``EXTERNAL-IP`` value of the ingress load balancer service
ksqldb.$DOMAIN         The ``EXTERNAL-IP`` value of the ingress load balancer service
controlcenter.$DOMAIN  The ``EXTERNAL-IP`` value of the ingress load balancer service
====================== ===============================================================

If you want to access MDS externally, add the following line to the /etc/hosts as well

::
      
  127.0.1.1       mds.confluent.edu

=================================================
Create RBAC Rolebindings for Control Center admin
=================================================

Create Control Center Role Binding for a Control Center ``testadmin`` user.

::

  kubectl apply -f $TUTORIAL_HOME/controlcenter-testadmin-rolebindings.yaml --namespace confluent

========
Validate
========

Validate in Control Center
^^^^^^^^^^^^^^^^^^^^^^^^^^

Use Control Center to monitor the Confluent Platform, and see the created topic
and data. You can visit the external URL you set up for `Control Center <https://controlcenter.confluent.edu:8443>`__, or visit the URL
through a local port forwarding like below:

#. Set up port forwarding to Control Center web UI from local machine:

   ::

     kubectl port-forward controlcenter-0 9021:9021 --namespace confluent

#. Browse to Control Center. You will log in as the ``testadmin`` user, with ``testadmin`` password.

   ::
   
     https://localhost:9021

The ``testadmin`` user (``testadmin`` password) has the ``SystemAdmin`` role granted and will have access to the
cluster and broker information.



=========
Tear down
=========

::

  kubectl delete ingressroutetcp/ingress-all
  kubectl delete confluentrolebinding --all --namespace confluent
  kubectl delete -f /home/training/confluent-operator/networking/external-access-static-host-based/kafka-bootstrap-service.yaml
  kubectl delete -f /home/training/confluent-operator/networking/external-access-static-host-based/connect-bootstrap-service.yaml
  kubectl delete -f /home/training/confluent-operator/networking/external-access-static-host-based/ksqldb-bootstrap-service.yaml
  kubectl delete -f /home/training/confluent-operator/security/internal_external-tls_mtls_confluent-rbac/mds-bootstrap-service.yaml
  kubectl delete -f $TUTORIAL_HOME/confluent-platform-production.yaml --namespace confluent
  kubectl delete secret rest-credential ksqldb-mds-client sr-mds-client connect-mds-client krp-mds-client c3-mds-client mds-client ca-pair-sslcerts --namespace confluent
  kubectl delete secret mds-token --namespace confluent
  kubectl delete secret credential --namespace confluent
  kubectl delete secret tls-group1 --namespace confluent
  helm delete test-ldap --namespace confluent
  helm delete operator --namespace confluent


=====================================
Appendix: Update authentication users
=====================================

In order to add users to the authenticated users list, you'll need to update the list in the following files:

- For Kafka users, update the list in ``creds-kafka-sasl-users.json``.
- For Control Center users, update the list in ``creds-control-center-users.txt``.

After updating the list of users, you'll update the Kubernetes secret.

::

  kubectl --namespace confluent create secret generic credential \
      --from-file=plain-users.json=$TUTORIAL_HOME/creds-kafka-sasl-users.json \
      --from-file=digest-users.json=$TUTORIAL_HOME/creds-zookeeper-sasl-digest-users.json \
      --from-file=digest.txt=$TUTORIAL_HOME/creds-kafka-zookeeper-credentials.txt \
      --from-file=plain.txt=$TUTORIAL_HOME/creds-client-kafka-sasl-user.txt \
      --from-file=basic.txt=$TUTORIAL_HOME/creds-control-center-users.txt \
      --from-file=ldap.txt=$TUTORIAL_HOME/ldap.txt \
      --save-config --dry-run=client -oyaml | kubectl apply -f -

In this above CLI command, you are generating the YAML for the secret, and applying it as an update to the existing secret ``credential``.

There's no need to restart the Kafka brokers or Control Center. The updates users list is picked up by the services.
