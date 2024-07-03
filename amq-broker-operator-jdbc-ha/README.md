# amq-broker-operator-jdbc-ha

## Init
```
mkdir -p workspace
wget -O workspace/apache-artemis-2.34.0-bin.zip https://archive.apache.org/dist/activemq/activemq-artemis/2.34.0/apache-artemis-2.34.0-bin.zip
unzip workspace/apache-artemis-2.34.0-bin.zip -d workspace/apache-artemis-2.34.0-bin
mv workspace/apache-artemis-2.34.0-bin/apache-artemis-2.34.0 workspace/apache-artemis-2.34.0
rm -rf workspace/apache-artemis-2.34.0-bin*
```

## Openshift
Log in to your server by using the command `oc login` and create new project y using the command `oc new-project`, i.e.
```
oc login --token=sha256~n1eCPRPn2jSgVXJU8ObmfmvRqlIfFxz-MjJ2f6WnYqM --server=https://a603c8cd206bf4fe98f4d3817cda2dda-011da12348812db5.elb.us-east-1.amazonaws.com:6443
oc new-project oracle-jdbc-shared-store
```

## Oracle Database
```
oc apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: oracle-database-deployment
  namespace: default
spec:
  selector:
    matchLabels:
      app: oracle-database
  replicas: 1
  template:
    metadata:
      labels:
        app: oracle-database
    spec:
      containers:
        - name: oracle-database-container
          image: container-registry.oracle.com/database/free:23.3.0.0
          env:
            - name: ORACLE_PWD
              value: secret
          resources:
            limits:
              cpu: 2
              memory: 4Gi
          ports:
            - containerPort: 1521
          readinessProbe:
            tcpSocket:
              port: 1521
            initialDelaySeconds: 15
            periodSeconds: 10
---
apiVersion: v1
kind: Service
metadata:
  name: oracle-database-service
  namespace: default
spec:
  selector:
    app: oracle-database
  ports:
    - name: oracle-database-port
      protocol: TCP
      port: 1521
      targetPort: 1521
EOF
```

## Certificates
```
wget -O workspace/server-keystore.jks https://github.com/apache/activemq-artemis/raw/main/tests/security-resources/server-keystore.jks
wget -O workspace/server-ca-truststore.jks https://github.com/apache/activemq-artemis/raw/main/tests/security-resources/server-ca-truststore.jks
wget -O workspace/client-ca-truststore.jks https://github.com/apache/activemq-artemis/raw/main/tests/security-resources/client-ca-truststore.jks
oc create secret generic ext-acceptor-ssl-secret \
--from-file=broker.ks=workspace/server-keystore.jks \
--from-file=client.ts=workspace/client-ca-truststore.jks \
--from-literal=keyStorePassword=securepass \
--from-literal=trustStorePassword=securepass
```

## AMQ Broker Operator
```
oc apply -f - <<EOF
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: amq-broker-rhel8
  namespace: openshift-operators
spec:
  channel: 7.12.x
  installPlanApproval: Automatic
  name: amq-broker-rhel8
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF
```

## ActiveMQ Artemis
```
oc apply -f - <<EOF
apiVersion: broker.amq.io/v1beta1
kind: ActiveMQArtemis
metadata: 
  name: peer-broker-a
spec:
  deploymentPlan:
    size: 1
    clustered: false
    persistenceEnabled: false
    labels:
      peer.group: jdbc-ha
    livenessProbe:
      exec:
        command:
        - test
        - -f
        - /home/jboss/amq-broker/lock/cli.lock
  env:
    - name: ARTEMIS_EXTRA_LIBS
      value: '/amq/init/config/extra-libs'
  brokerProperties:
    - 'criticalAnalyser=false'
    - 'storeConfiguration=DATABASE'
    - 'storeConfiguration.jdbcDriverClassName=oracle.jdbc.OracleDriver'
    - 'storeConfiguration.jdbcConnectionUrl=jdbc:oracle:thin:SYSTEM/secret@oracle-database-service.default.svc.cluster.local:1521/FREEPDB1'
    - 'storeConfiguration.jdbcLockRenewPeriodMillis=2000'
    - 'storeConfiguration.jdbcLockExpirationMillis=6000'
    - 'HAPolicyConfiguration=SHARED_STORE_PRIMARY'
  acceptors:
  - name: ext-acceptor
    protocols: CORE
    port: 61626
    expose: true
    sslEnabled: true
    sslSecret: ext-acceptor-ssl-secret
  console:
    expose: true
  resourceTemplates:
    - selector:
        kind: StatefulSet
      patch:
        kind: StatefulSet
        spec:
          template:
            spec:
              initContainers:
                - name: oracle-database-jdbc-driver-init
                  image: registry.redhat.io/amq7/amq-broker-rhel8:7.12
                  volumeMounts:
                    - name: amq-cfg-dir
                      mountPath: /amq/init/config
                  command:
                    - "bash"
                    - "-c"
                    - "mkdir -p /amq/init/config/extra-libs && curl -Lo /amq/init/config/extra-libs/ojdbc11.jar https://download.oracle.com/otn-pub/otn_software/jdbc/233/ojdbc11.jar"
---
apiVersion: broker.amq.io/v1beta1
kind: ActiveMQArtemis
metadata: 
  name: peer-broker-b
spec:
  deploymentPlan:
    size: 1
    clustered: false
    persistenceEnabled: false
    labels:
      peer.group: jdbc-ha
    livenessProbe:
      exec:
        command:
        - test
        - -f
        - /home/jboss/amq-broker/lock/cli.lock
  env:
    - name: ARTEMIS_EXTRA_LIBS
      value: '/amq/init/config/extra-libs'
  brokerProperties:
    - 'criticalAnalyser=false'
    - 'storeConfiguration=DATABASE'
    - 'storeConfiguration.jdbcDriverClassName=oracle.jdbc.OracleDriver'
    - 'storeConfiguration.jdbcConnectionUrl=jdbc:oracle:thin:SYSTEM/secret@oracle-database-service.default.svc.cluster.local:1521/FREEPDB1'
    - 'storeConfiguration.jdbcLockRenewPeriodMillis=2000'
    - 'storeConfiguration.jdbcLockExpirationMillis=6000'
    - 'HAPolicyConfiguration=SHARED_STORE_PRIMARY'
  acceptors:
  - name: ext-acceptor
    protocols: CORE
    port: 61626
    expose: true
    sslEnabled: true
    sslSecret: ext-acceptor-ssl-secret
  console:
    expose: true
  resourceTemplates:
    - selector:
        kind: StatefulSet
      patch:
        kind: StatefulSet
        spec:
          template:
            spec:
              initContainers:
                - name: oracle-database-jdbc-driver-init
                  image: registry.redhat.io/amq7/amq-broker-rhel8:7.12
                  volumeMounts:
                    - name: amq-cfg-dir
                      mountPath: /amq/init/config
                  command:
                    - "bash"
                    - "-c"
                    - "mkdir -p /amq/init/config/extra-libs && curl -Lo /amq/init/config/extra-libs/ojdbc11.jar https://download.oracle.com/otn-pub/otn_software/jdbc/233/ojdbc11.jar"
EOF
```
## ext-acceptor-svc
```
oc apply -f - <<EOF
apiVersion: v1
kind: Service
metadata: 
  name: ext-acceptor-svc
spec:
  ports:
    - protocol: TCP
      port: 61626
      targetPort: 61626
  selector:
    peer.group: jdbc-ha
  type: ClusterIP
  sessionAffinity: None
  publishNotReadyAddresses: true
---
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: ext-acceptor-svc-rte
spec:
  port:
    targetPort: 61626
  tls:
    termination: passthrough 
    insecureEdgeTerminationPolicy: None 
  to:
    kind: Service
    name: ext-acceptor-svc
EOF
```

## Hosts
```
export EXT_ACCEPTOR_HOST=$(oc get route ext-acceptor-svc-rte -o json | jq -r '.spec.host')
export PEER_BROKER_A_EXT_ACCEPTOR_HOST=$(oc get route peer-broker-a-ext-acceptor-0-svc-rte -o json | jq -r '.spec.host')
export PEER_BROKER_A_CONSOLE_HOST=$(oc get route peer-broker-a-wconsj-0-svc-rte -o json | jq -r '.spec.host')
export PEER_BROKER_B_EXT_ACCEPTOR_HOST=$(oc get route peer-broker-b-ext-acceptor-0-svc-rte -o json | jq -r '.spec.host')
export PEER_BROKER_B_CONSOLE_HOST=$(oc get route peer-broker-b-wconsj-0-svc-rte -o json | jq -r '.spec.host')
```

## Producer
```
workspace/apache-artemis-2.34.0/bin/artemis producer --verbose --destination queue://TEST --user admin --password admin --protocol core --sleep 1000 --url "tcp://${EXT_ACCEPTOR_HOST}:443?sslEnabled=true&verifyHost=false&trustStorePath=workspace/server-ca-truststore.jks&trustStorePassword=securepass&useTopologyForLoadBalancing=false&initialConnectAttempts=-1&failoverAttempts=-1"
```

## Consumer
```
workspace/apache-artemis-2.34.0/bin/artemis consumer --verbose --destination queue://TEST --user admin --password admin --protocol core --sleep 1000 --url "tcp://${EXT_ACCEPTOR_HOST}:443?sslEnabled=true&verifyHost=false&trustStorePath=workspace/server-ca-truststore.jks&trustStorePassword=securepass&useTopologyForLoadBalancing=false&initialConnectAttempts=-1&failoverAttempts=-1"
```

## Test
```
oc patch  ActiveMQArtemis peer-broker-a --type='json' -p='[{"op": "replace", "path": "/spec/deploymentPlan/size", "value":0}]'

oc patch  ActiveMQArtemis peer-broker-a --type='json' -p='[{"op": "replace", "path": "/spec/deploymentPlan/size", "value":1}]'

oc patch  ActiveMQArtemis peer-broker-b --type='json' -p='[{"op": "replace", "path": "/spec/deploymentPlan/size", "value":0}]'

oc patch  ActiveMQArtemis peer-broker-b --type='json' -p='[{"op": "replace", "path": "/spec/deploymentPlan/size", "value":1}]'

workspace/apache-artemis-2.34.0/bin/artemis check queue --name TEST --produce 10 --browse 10 --consume 10 --url "tcp://${PEER_BROKER_A_EXT_ACCEPTOR_HOST}:443?name=artemis&useTopologyForLoadBalancing=false&sslEnabled=true&verifyHost=false&trustStorePath=workspace/server-ca-truststore.jks&trustStorePassword=securepass"

workspace/apache-artemis-2.34.0/bin/artemis check queue --name TEST --produce 10 --browse 10 --consume 10 --url "tcp://${PEER_BROKER_B_EXT_ACCEPTOR_HOST}:443?name=artemis&useTopologyForLoadBalancing=false&sslEnabled=true&verifyHost=false&trustStorePath=workspace/server-ca-truststore.jks&trustStorePassword=securepass"

workspace/apache-artemis-2.34.0/bin/artemis producer --verbose --destination queue://TEST --user admin --password admin --protocol core --sleep 1000 --url "(tcp://${PEER_BROKER_A_EXT_ACCEPTOR_HOST}:443,tcp://${PEER_BROKER_B_EXT_ACCEPTOR_HOST}:443)?sslEnabled=true&verifyHost=false&trustStorePath=workspace/server-ca-truststore.jks&trustStorePassword=securepass&useTopologyForLoadBalancing=false&initialConnectAttempts=-1&failoverAttempts=-1"

workspace/apache-artemis-2.34.0/bin/artemis consumer --verbose --destination queue://TEST --user admin --password admin --protocol core --sleep 1000 --url "(tcp://${PEER_BROKER_A_EXT_ACCEPTOR_HOST}:443,tcp://${PEER_BROKER_B_EXT_ACCEPTOR_HOST}:443)?sslEnabled=true&verifyHost=false&trustStorePath=workspace/server-ca-truststore.jks&trustStorePassword=securepass&useTopologyForLoadBalancing=false&initialConnectAttempts=-1&failoverAttempts=-1"
```
