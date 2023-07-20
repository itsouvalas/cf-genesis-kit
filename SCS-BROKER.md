# Spring Cloud Services Broker

[scs-broker](https://github.com/cloudfoundry-community/scs-broker) is a Service Broker that once deployed, allows the creation of two services:

* [cf-spring-cloud-config-server](https://github.com/cloudfoundry-community/cf-spring-cloud-config-server)
* [scs-service-registry](https://github.com/cloudfoundry-community/scs-service-registry)

## Deploy the [scs-broker](https://github.com/cloudfoundry-community/scs-broker)

In order to deploy the scs-broker you first need to enable the feature on your cf-genesis-kit

```yaml
---
kit:
  name:    cf
  version: 2.3.1
  features:
    ...
    - scs-integration
```

This will allow [scs](overlay/addons/scs.yml) addon to be added and enable the scs-broker to integrate with uaa.

Once the feature has been enabled the environment re-deployed, you can deploy the scs-broker by issuing at the root of your cf-deployment directory:

```bash
genesis do environment-name.yml scs deploy
```

This will create the service broker under the `scs` space as an application:

```bash
Getting apps in org system / space scs as admin...

name                                                    requested state   processes           routes
scs-broker                                              started           web:1/1             scs-broker.run.it-lab.instantpc.support
```

Before you are able to use the service broker you ought to register it first by issuing:

```bash
genesis do environment-name.yml scs register
```

Once registered you will be able to review it using `cf service-brokers`

```bash
Getting service brokers as admin...
name                        url
scs-broker                  https://scs-broker.run.it-lab.instantpc.support
```

With the service broker deployed and register you can now enable the access to the corresponding services:

```bash
cf enable-service-access service-registry -b scs-broker -p default -o system
```

```bash
cf enable-service-access config-server -b scs-broker -p default -o system
```

You are now ready to start creating the corresponding services.

## config-server

The [Spring Cloud Config Server](https://cloud.spring.io/spring-cloud-config/multi/multi__spring_cloud_config_server.html) _provides an HTTP resource-based API for external configuration (name-value pairs or equivalent YAML content)_. In order fo the service to be instantiated you need to provide a config file in `json` format so that it serves the corresponding properties of the spring cloud app that uses it:

`configtest-config.json`

```json
{
  "forward-headers-strategy": "framework",
  "health": {
    "enabled": false
  },
  "git": {
    "uri": "https://github.com/itsouvalas/sc-app-config.git"
          }
}
```

in the above example we are using `https://github.com/itsouvalas/sc-app-config.git` as the repository that hold the properties we would like to share to our application:

`config-test-development.properties`

```bash
message=Success! - You have succesfully configured your Spring Cloud App to use the config server
```

to create the service instance issue the following:

```bash
cf create-service config-server default config-server -c configtest-config.json
```

### Important
The `scs-broker` along with its services `service-registry` and `config-server`, are all run as spring applications under the `scs` space. Depending on your infrastucture those might take some time before they are actually serving traffic. To avoid chasing imaginary dragons, make sure that you allow ample time before interacting with them, or use `cf logs app-name --recent` to confirm that they have finished running.

### Testing config-server

Once you have created the service you are ready to query for the properties you are after. Use `cf apps` to confirm the status of the applications and their deployed routes:

```bash
Getting apps in org system / space scs as admin...

name                                                    requested state   processes           routes
config-server-cb108deb-d302-4d2b-9fa0-0b9744cdf00f      started           web:1/1, task:0/0   config-server-cb108deb-d302-4d2b-9fa0-0b9744cdf00f.run.it-lab.instantpc.support
scs-broker                                              started           web:1/1             scs-broker.run.it-lab.instantpc.support
```

Based on the name of the application, the profile, and the branch the properties have been added, you can query the service broker for the property in question:

app-name: configtest
profile: development
branch: main

```bash
curl -k https://config-server-cb108deb-d302-4d2b-9fa0-0b9744cdf00f.run.it-lab.instantpc.support/config-test/development/main
```
```json
{"name":"config-test","profiles":["development"],"label":"main","version":null,"state":null,"propertySources":[{"name":"credhub-config-test-development-main","source":{}},{"name":"https://github.com/itsouvalas/sc-app-config.git/config-test-development.properties","source":{"message":"Success! - You have succesfully configured your Spring Cloud App to use the config server"}}]}
```

