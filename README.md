# AMQ Broker

## Install Operator

Install operator AMQ Broker for RHEL8 from OperatorHUB.

## Install AMQ Broker
```bash
oc new-project test-upgrade-amqbroker
```

```bash
oc apply -f ActiveMQArtemis.yml -n test-upgrade-amqbroker
```

## Tests

### Producer
```bash
oc rsh ex-aao-ss-0 /home/jboss/amq-broker/bin/artemis producer \
	--url tcp://ex-aao-client-0-svc.test-upgrade-amqbroker.svc.cluster.local:5673 \
	--message-count 100 \
	--protocol AMQP \
	--destination queue://demoQueue
```

### Consumer
```bash
oc rsh ex-aao-ss-0 /home/jboss/amq-broker/bin/artemis consumer \
	--url tcp://ex-aao-client-0-svc.test-upgrade-amqbroker.svc.cluster.local:5673 \
	--protocol AMQP \
	--destination queue://demoQueue \
	--message-count 100
```

### Queue stat
```bash
oc rsh ex-aao-ss-0 /home/jboss/amq-broker/bin/artemis queue stat \
	--url tcp://ex-aao-hdls-svc.test-upgrade-amqbroker.svc.cluster.local:61616 \
	--clustered
```

## Upgrade operator

Uninstall operator `AMQ Broker for RHEL8` 7.12.x from OperatorHUB and then install new operator `AMQ Broker for RHEL9` 7.13.1 from OperatorHUB.

> [!WARNING]
> If the version in the custom resource `ActiveMQArtemis` has not been set, the broker will also be automatically updated!

## Upgrade broker from 7.12.x to 7.13.1

**Patch CR version**
```bash
oc patch ActiveMQArtemis ex-aao --type='merge' -p '{"spec": {"version": "7.13.1"}}'
```

## Upgrade broker from 7.12.5 to 7.13.1

**Patch CR version**
```bash
oc patch ActiveMQArtemis ex-aao --type='merge' -p '{"spec": {"version": "7.13.1"}}'
```

> [!CAUTION] 
> Operator logs report the following error: 

```
ERROR	util_common	failed to determine broker version from cr	{"error": "did not find a matching broker in the supported list for 7.12.5"...
```

Because the version 7.12.5 not in the supported list as describe [here](https://docs.redhat.com/it/documentation/red_hat_amq_broker/7.13/html/deploying_amq_broker_on_openshift/assembly_br-upgrading-operator-based-broker-deployments_broker-ocp#con-br-upgrading-operator-operatorhub-broker-images_broker-ocp).

## Docs
1. https://docs.redhat.com/it/documentation/red_hat_amq_broker/7.13/html/deploying_amq_broker_on_openshift/assembly_br-upgrading-operator-based-broker-deployments_broker-ocp#proc-br-upgrading-operator-operatorhub-pre7100to7110_broker-ocp
2. https://access.redhat.com/solutions/7120341


## Logging

```bash
oc cp ex-aao-ss-0:/home/jboss/amq-broker/etc/log4j2.properties logging.properties -c ex-aao-container
```

Adapt the log4j2 config file logging.properties and then

```bash
oc create configmap newlog4j-logging-config --from-file=log/logging.properties --dry-run=client -o yaml > log/newlog4j-logging-config.yml
```

```bash
oc apply -f log/newlog4j-logging-config.yml
```

```bash
oc patch ActiveMQArtemis ex-aao --type='merge' -p '{"spec": {"deploymentPlan": {"extraMounts": {"configMaps": ["newlog4j-logging-config"]}}}}'
```

The operator reconfigures the broker pods to use the new logging.properties file taken by the referenced newlog4j-logging-config cm. Future changes to the newlog4j-logging-config cm will be used automatically by the broker pods.
Checking the pod yaml definition, the referenced cm is mounted under the path /amq/extra/configmaps/newlog4j-logging-config and the logging settings is in the env property JAVA_ARGS_APPEND with value '-Dlog4j2.configurationFile=/amq/extra/configmaps/newlog4j-logging-config/logging.properties'

Examples of audit.resource:
```bash
2025-10-14 12:06:41,438 INFO [org.apache.activemq.audit.resource] AMQ601767: CORE connection 188f548c for user unknown@10.131.0.42:48774 created
2025-10-14 12:06:41,499 INFO [org.apache.activemq.audit.resource] AMQ601715: User admin(admin)@10.131.0.42:48774 successfully authenticated on connection 188f548c
2025-10-14 12:06:41,618 INFO [org.apache.activemq.audit.resource] AMQ601768: CORE connection 188f548c for user admin(admin)@10.131.0.42:48774 destroyed
```

Examples of audit.base (extremely verbose):
```bash
2025-10-14 12:06:41,499 INFO [org.apache.activemq.audit.base] AMQ601267: User admin(admin)@10.131.0.42:48774 is creating a core session on target resource ActiveMQServerImpl::name=amq-broker with parameters: [3ea64306-a8f6-11f0-9ce9-0a580a83002a, null, ****, 102400, RemotingConnectionImpl [ID=188f548c, clientID=null, nodeID=bcd6de51-a8f4-11f0-854e-0a580a83002a, transportConnection=org.apache.activemq.artemis.core.remoting.impl.netty.NettyServerConnection@74d42ae2[ID=188f548c, local= /10.131.0.42:61616, remote=/10.131.0.42:48774]], true, true, false, false, null, org.apache.activemq.artemis.core.protocol.core.impl.CoreSessionCallback@7a13da08, true, {}]
2025-10-14 12:06:41,523 INFO [org.apache.activemq.audit.base] AMQ601171: User admin(admin)@10.131.0.42:48774 is getting ID on target resource: QueueImpl[name=$.artemis.internal.sf.my-cluster.afc81f71-a8f4-11f0-882d-0a580a830029, postOffice=PostOfficeImpl [server=ActiveMQServerImpl::name=amq-broker], temp=false]@1b756772
2025-10-14 12:06:41,524 INFO [org.apache.activemq.audit.base] AMQ601014: User admin(admin)@10.131.0.42:48774 is getting name on target resource: QueueImpl[name=$.artemis.internal.sf.my-cluster.afc81f71-a8f4-11f0-882d-0a580a830029, postOffice=PostOfficeImpl [server=ActiveMQServerImpl::name=amq-broker], temp=false]@1b756772
2025-10-14 12:06:41,524 INFO [org.apache.activemq.audit.base] AMQ601015: User admin(admin)@10.131.0.42:48774 is getting address on target resource: QueueImpl[name=$.artemis.internal.sf.my-cluster.afc81f71-a8f4-11f0-882d-0a580a830029, postOffice=PostOfficeImpl [server=ActiveMQServerImpl::name=amq-broker], temp=false]@1b756772
2025-10-14 12:06:41,524 INFO [org.apache.activemq.audit.base] AMQ601016: User admin(admin)@10.131.0.42:48774 is getting filter on target resource: QueueImpl[name=$.artemis.internal.sf.my-cluster.afc81f71-a8f4-11f0-882d-0a580a830029, postOffice=PostOfficeImpl [server=ActiveMQServerImpl::name=amq-broker], temp=false]@1b756772
2025-10-14 12:06:41,524 INFO [org.apache.activemq.audit.base] AMQ601017: User admin(admin)@10.131.0.42:48774 is getting durable property on target resource: QueueImpl[name=$.artemis.internal.sf.my-cluster.afc81f71-a8f4-11f0-882d-0a580a830029, postOffice=PostOfficeImpl [server=ActiveMQServerImpl::name=amq-broker], temp=false]@1b756772
2025-10-14 12:06:41,524 INFO [org.apache.activemq.audit.base] AMQ601214: User admin(admin)@10.131.0.42:48774 is getting paused property on target resource: QueueImpl[name=$.artemis.internal.sf.my-cluster.afc81f71-a8f4-11f0-882d-0a580a830029, postOffice=PostOfficeImpl [server=ActiveMQServerImpl::name=amq-broker], temp=false]@1b756772
```

Examples of audit.message:
```bash
2025-10-14 12:06:41,500 INFO [org.apache.activemq.audit.message] AMQ601502: User anonymous@10.131.0.41:37686 acknowledged message from notif.e2cc45f7-a8f4-11f0-882d-0a580a830029.ActiveMQServerImpl_name=amq-broker: CoreMessage[messageID=290, durable=true, userID=null, priority=0, timestamp=Tue Oct 14 12:06:41 UTC 2025, expiration=0, durable=true, address=activemq.notifications, size=605, properties=TypedProperties[_AMQ_User=NULL-value, _AMQ_Distance=0, _AMQ_SessionName=3ea64306-a8f6-11f0-9ce9-0a580a83002a, _AMQ_Protocol_Name=CORE, _AMQ_Address=activemq.notifications, _AMQ_NotifType=SESSION_CREATED, _AMQ_Client_ID=NULL-value, _AMQ_NotifTimestamp=1760443601500, _AMQ_ConnectionName=188f548c]]@1897961230, transaction: null
2025-10-14 12:06:41,500 INFO [org.apache.activemq.audit.message] AMQ601501: User anonymous@10.131.0.41:37686 is consuming a message from notif.e2cc45f7-a8f4-11f0-882d-0a580a830029.ActiveMQServerImpl_name=amq-broker: Reference[290]:RELIABLE:CoreMessage[messageID=290, durable=true, userID=null, priority=0, timestamp=Tue Oct 14 12:06:41 UTC 2025, expiration=0, durable=true, address=activemq.notifications, size=605, properties=TypedProperties[_AMQ_User=NULL-value, _AMQ_Distance=0, _AMQ_SessionName=3ea64306-a8f6-11f0-9ce9-0a580a83002a, _AMQ_Protocol_Name=CORE, _AMQ_Address=activemq.notifications, _AMQ_NotifType=SESSION_CREATED, _AMQ_Client_ID=NULL-value, _AMQ_NotifTimestamp=1760443601500, _AMQ_ConnectionName=188f548c]]@1897961230
2025-10-14 12:06:32,213 INFO [org.apache.activemq.audit.message] AMQ601501: User admin(admin)@169.254.0.5:38026 is consuming a message from demoQueue: Reference[62]:RELIABLE:AMQPStandardMessage( [durable=true, messageID=62, address=demoQueue, size=218, scanningStatus=SCANNED, applicationProperties={ThreadSent=Producer demoQueue, thread=0, count=6}, messageAnnotations={x-opt-jms-dest=0, x-opt-jms-msg-type=5}, properties=Properties{messageId=ID:9d0f6448-450b-43c9-ab50-95e8233d3ca8:2:1:1-7, userId=null, to='demoQueue', subject='null', replyTo='null', correlationId=null, contentType=null, contentEncoding=null, absoluteExpiryTime=null, creationTime=Tue Oct 14 12:06:21 UTC 2025, groupId='null', groupSequence=null, replyToGroupId='null'}, extraProperties = null]
2025-10-14 12:06:32,322 INFO [org.apache.activemq.audit.message] AMQ601759: User admin(admin)@169.254.0.5:38026 added acknowledgement of a message from demoQueue: AMQPStandardMessage( [durable=true, messageID=62, address=demoQueue, size=218, scanningStatus=SCANNED, applicationProperties={ThreadSent=Producer demoQueue, thread=0, count=6}, messageAnnotations={x-opt-jms-dest=0, x-opt-jms-msg-type=5}, properties=Properties{messageId=ID:9d0f6448-450b-43c9-ab50-95e8233d3ca8:2:1:1-7, userId=null, to='demoQueue', subject='null', replyTo='null', correlationId=null, contentType=null, contentEncoding=null, absoluteExpiryTime=null, creationTime=Tue Oct 14 12:06:21 UTC 2025, groupId='null', groupSequence=null, replyToGroupId='null'}, extraProperties = null] to transaction: TransactionImpl [xid=null, txID=168, xid=null, state=ACTIVE, createTime=1760443592320(Tue Oct 14 12:06:32 UTC 2025), timeoutSeconds=-1, nr operations = 2]@579a0bc3
2025-10-14 12:06:32,399 INFO [org.apache.activemq.audit.message] AMQ601502: User admin(admin)@169.254.0.5:38026 acknowledged message from demoQueue: AMQPStandardMessage( [durable=true, messageID=62, address=demoQueue, size=218, scanningStatus=SCANNED, applicationProperties={ThreadSent=Producer demoQueue, thread=0, count=6}, messageAnnotations={x-opt-jms-dest=0, x-opt-jms-msg-type=5}, properties=Properties{messageId=ID:9d0f6448-450b-43c9-ab50-95e8233d3ca8:2:1:1-7, userId=null, to='demoQueue', subject='null', replyTo='null', correlationId=null, contentType=null, contentEncoding=null, absoluteExpiryTime=null, creationTime=Tue Oct 14 12:06:21 UTC 2025, groupId='null', groupSequence=null, replyToGroupId='null'}, extraProperties = null], transaction: TransactionImpl [xid=null, txID=168, xid=null, state=COMMITTED, createTime=1760443592320(Tue Oct 14 12:06:32 UTC 2025), timeoutSeconds=-1, nr operations = 0]@579a0bc3
```