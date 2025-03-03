# Installation

Trino Gateway is distributed as an executable JAR file. The [release
notes](release-notes.md) contain links to download specific versions.
Alternatively, you can look at the [development instructions](development.md) to
build the JAR file or use the TrinoGatewayRunner for local testing.
The [quickstart guide](quickstart.md) contains instructions for running the
application locally. 

Following are instructions for installing Trino Gateway for production
environments.

## Requirements

Consider the following requirements for your Trino Gateway installation.

### Java

Trino Gateway requires a Java 22 runtime. Older versions of Java can not be
used. Newer versions might work but are not tested.

Verify the Java version on your system with `java -version`.

### Operating system

No specific operating system is required. All testing and development is
performed with Linux and MacOS.

### Processor architecture

No specific processor architecture is required, as long as a suitable Java
distribution is installed.  

### Backend database

Trino Gateway requires a MySQL or PostgreSQL database.

Use the following scripts in the `gateway-ha/src/main/resources/` folder to
initialize the database:

* `gateway-ha-persistence-mysql.sql` for MySQL
* `gateway-ha-persistence-postgres.sql` for PostgreSQL

The files are also included in the JAR file.

### Trino clusters

The proxied Trino clusters behind the Trino Gateway must support the Trino JDBC
driver and the Trino REST API for cluster and node health information.
Typically, this means that Trino versions 354 and higher should work, however
newer Trino versions are strongly recommended.

Trino-derived projects and platforms may work if the Trino JDBC driver and the
REST API are supported. For example, Starburst Galaxy and Starburst Enterprise
are known to work. Trino deployments with the Helm chart and other means on
various cloud platforms, such as Amazon EKS also work. However Amazon Athena
does not work since it uses alternative, custom protocols and lacks the concept
of individual clusters.

### Trino configuration

From a users perspective Trino Gateway acts as a transparent proxy for one 
or more Trino clusters. The following Trino configuration tips should be 
taken into account for all clusters behind the Trino Gateway.

If all client and server communication is routed through Trino Gateway, 
then process forwarded HTTP headers must be enabled:

```commandline
http-server.process-forwarded=true
```

Without this setting, first requests go from the user to Trino Gateway and then
to Trino correctly. However, the URL for subsequent next URIs for more results
in a query provided by Trino is then using the local URL of the Trino cluster,
and not the URL of the Trino Gateway. This circumvents the Trino Gateway for all
these requests. In scenarios, where the local URL of the Trino cluster is private 
to the Trino cluster on the network level, these following calls do not work
at all for users.

This setting is also required for Trino to authenticate in the case TLS is 
terminated at the Trino Gateway. Normally it refuses to authenticate plain HTTP 
requests, but if `http-server.process-forwarded=true` it authenticates over 
HTTP if the request includes `X-Forwarded-Proto: HTTPS`.

To prevent Trino Gateway from sending `X-Forwarded-*` headers, add the following configuration:

```yaml
routing:
  addXForwardedHeaders: false
```

Find more information in [the related Trino documentation](https://trino.io/docs/current/security/tls.html#use-a-load-balancer-to-terminate-tls-https).

## Configuration

After downloading or building the JAR, rename it to `gateway-ha.jar`,
and place it in a directory with read and write access such as `/opt/trinogateway`.

Copy the example config file `gateway-ha-config.yml` from the `gateway-ha/`
directory into the same directory, and update the configuration as needed.

Each component of the Trino Gateway has a corresponding node in the
configuration YAML file.

### Configure routing rules

Find more information in the [routing rules documentation](routing-rules.md).

### Configure logging <a name="logging">

To configure the logging level for various classes, specify the path to the 
`log.properties` file by setting `log.levels-file` in `serverConfig`.

For additional configurations, use the `log.*` properties from the 
[Trino logging properties documentation](https://trino.io/docs/current/admin/properties-logging.html) and specify
the properties in `serverConfig`.

### Proxying additional paths

By default, Trino Gateway only proxies requests to paths starting with
`/v1/statement`, `/v1/query`, `/ui`, `/v1/info`, `/v1/node`, `/ui/api/stats` and
`/oauth`.

If you want to proxy additional paths, you can add them by adding the
`extraWhitelistPaths` node to your configuration YAML file.
Trino Gateway takes regexes from `extraWhitelistPaths` and forwards only
those requests with a URI that exactly match. Be sure
to use single-quoted strings so that escaping is not required.

```yaml
extraWhitelistPaths:
  - '/ui/insights'
  - '/api/v1/biac'
  - '/api/v1/dataProduct'
  - '/api/v1/dataproduct'
  - '/api/v2/.*'
  - '/ext/faster'
```

### Configure additional v1/statement-like paths

The Trino client protocol specifies that queries are initiated by a POST to `v1/statement`. 
The Trino Gateway incorporates this into its routing logic by extracting and recording the 
query id from responses to such requests. If you use an experimental or commercial build of
Trino that supports additional endpoints, you can cause Trino Gateway to treat them 
equivalently to `/v1/statement` by adding them under the `additionalStatementPaths`
configuration node. They must be absolute, and no path can be a prefix to any other path.
The standard `/v1/statement` path is always included and does not need to be configured. 
For example:

```yaml
additionalStatementPaths:
  - '/ui/api/insights/ide/statement'
  - '/v2/statement'
```

## Configure behind a load balancer

A possible deployment of Trino Gateway is to run multiple instances of Trino 
Gateway behind another generic load balancer, such as a load balancer from 
your cloud hosting provider. In this deployment you must configure the 
`serverConfig` to include enabling process forwarded HTTP headers:

```yaml
serverConfig:
  http-server.process-forwarded: true
```

## Running Trino Gateway

Start Trino Gateway with the following java command in the directory of the
JAR and YAML files:

```shell
java -XX:MinRAMPercentage=50 -XX:MaxRAMPercentage=80 \
    -jar gateway-ha.jar gateway-config.yml
```

### Helm

Helm manages the deployment of Kubernetes applications by templating Kubernetes
resources with a set of Helm charts. The Trino Gateway Helm chart consists 
of the following components:

* A `config` node for general configuration
* `dataStoreSecret`, `backendStateSecret` and `authenticationSecret` for 
  providing sensitive configurations through Kubernetes secrets, 
* Standard Helm options such as `replicaCount`, `resources` and `ingress`.

The default `values.yaml` found in the `helm/trino-gateway` folder includes
basic configuration options as an example. For a simple deployment, proceed with 
the following steps:

Create a yaml file containing the configuration for your `datastore`:

```shell
cat << EOF > datastore.yaml
dataStore:
   jdbcUrl: jdbc:postgresql://yourdatabasehost:5432/gateway
   user: postgres
   password: secretpassword
   driver: org.postgresql.Driver
EOF
```
Create a Kubernetes secret from this file:

```shell
kubectl create secret generic datastore-yaml --from-file datastore.yaml --dry-run=client -o yaml | kubectl apply -f -
```

Create a values override with a name such as `values-override.yaml` and
reference this secret in the `backendStateSecret` node:

```yaml
backendStateSecret:
    name: "datastore-yaml"
    key: "datastore.yaml"
```

When a Secret is created with the `--from-file` option, the filename is used as
the key. Finally, you can deploy Trino Gateway with the chart from the root 
of this repository:

```shell
helm install tg --values values-override.yaml helm/trino-gateway 
```

Secrets for `authenticationSecret` and `backendState` can be provisioned
similarly. Alternatively,  you can directly define the `config.backEndState` 
node in `values-override.yaml` and leave `backendStateSecret` undefined. 
However, a [Secret](https://kubernetes.
io/docs/concepts/configuration/secret/)
is recommended to protect the  database credentials required for this 
configuration.

#### Additional options

To implement routing rules, create a ConfigMap from your routing rules yaml
definition:

```shell
kubectl create cm routing-rules --from-file your-routing-rules.yaml
```

Then mount it to your container:

```yaml
volumes:
    - name: routing-rules
      configMap:
          name: routing-rules
          items:
              name: your-routing-rules.yaml
              path: your-routing-rules.yaml

volumeMounts:
    - name: routing-rules
      mountPath: "/etc/routing-rules/your-routing-rules.yaml"
      subPath: your-routing-rules.yaml
```

Ensure that the `mountPath` matches the `rulesConfigPath` specified in your
configuration. Note that the `subPath` is not strictly necessary, and if it 
is not specified the file is mounted at `mountPath/<configMap key>`. 
Kubernetes updates the mounted file when the ConfigMap is updated.

Standard Helm options such as `replicaCount`, `image`, `imagePullSecrets`, 
`service`, `ingress` and `resources` are supported. These are defined in 
`helm/values.yaml`. 

### Health Checks

Trino Gateway checks the health of each backend and **deactivates it if 
unhealthy**. A backend that fails a health check must be manually reset to 
active. Automatic recovery is not supported.

The type of health check is configured by setting

```yaml
clusterStatsConfiguration:
  monitorType: ""
```

to one of the following values.

#### INFO_API (default)

By default Trino Gateway uses the `v1/info` REST endpoint. A successful check is
defined as a 200 response with `starting: false`. Connection timeout parameters 
can be defined through the `monitor` node, for example

```yaml
monitor:
  connectTimeoutSeconds: 5
  requestTimeoutSeconds: 10
  idleTimeoutSeconds: 1
  retries: 1
```

All timeout parameters are optional.

#### JDBC

This uses a JDBC connection to query `system.runtime` tables for cluster 
information. It is required for the query count based routing strategy. This is
recommended over `UI_API` since it does not restrict the Web UI authentication
method of backend clusters. Configure a username and password by adding
`backendState` to your configuration. The username and password must be valid 
across all backends.

```yaml
backendState:
  username: "user"
  password: "password"
```

The request timeout can be set through

```yaml
monitor:
  requestTimeoutSeconds: 10
```

Other timeout parameters are not applicable to the JDBC connection.

#### UI_API

This pulls cluster information from the `ui/api/stats` REST endpoint. This is
supported for legacy reasons and may be deprecated in the future. It is only 
supported for backend clusters with `web-ui.authentication.type = PASSWORD`. Set
a username and password using `backendState` as with the `JDBC` option.

#### NOOP

This option disables health checks.
