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
