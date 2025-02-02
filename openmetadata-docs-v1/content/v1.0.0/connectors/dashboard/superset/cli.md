---
title: Run Superset Connector using the CLI
slug: /connectors/dashboard/superset/cli
---

# Run Superset using the metadata CLI

In this section, we provide guides and references to use the Superset connector.

Configure and schedule Superset metadata and profiler workflows from the OpenMetadata UI:

- [Requirements](#requirements)
- [Metadata Ingestion](#metadata-ingestion)

## Requirements

{%inlineCallout icon="description" bold="OpenMetadata 0.12 or later" href="/deployment"%}
To deploy OpenMetadata, check the Deployment guides.
{%/inlineCallout%}

To run the Ingestion via the UI you'll need to use the OpenMetadata Ingestion Container, which comes shipped with
custom Airflow plugins to handle the workflow deployment.

The ingestion also works with Superset 2.0.0 🎉

**Note:**

**API Connection**: To extract metadata from Superset via API, user must have at least `can read on Chart` & `can read on Dashboard` permissions.

**Database Connection**: To extract metadata from Superset via MySQL or Postgres database, database user must have at least `SELECT` priviledge on `dashboards` & `slices` tables within superset schema.


### Python Requirements

To run the Superset ingestion, you will need to install:

```bash
pip3 install "openmetadata-ingestion[superset]"
```

## Metadata Ingestion

All connectors are defined as JSON Schemas.
[Here](https://github.com/open-metadata/OpenMetadata/blob/main/openmetadata-spec/src/main/resources/json/schema/entity/services/connections/dashboard/supersetConnection.json)
you can find the structure to create a connection to Superset.

In order to create and run a Metadata Ingestion workflow, we will follow
the steps to create a YAML configuration able to connect to the source,
process the Entities if needed, and reach the OpenMetadata server.

The workflow is modeled around the following
[JSON Schema](https://github.com/open-metadata/OpenMetadata/blob/main/openmetadata-spec/src/main/resources/json/schema/metadataIngestion/workflow.json)

### 1. Define the YAML Config

This is a sample config for Superset:

{% codePreview %}

{% codeInfoContainer %}

#### Source Configuration - Service Connection

{% codeInfo srNumber=1 %}

**hostPort**: URL to the Superset instance.

**connection**: Add the connection details to fetch metadata from Superset either through APIs or Database.

#### For Superset API Connection:

**username**: Specify the User to connect to Superset. It should have enough privileges to read all the metadata.

**password**: Password for Superset.

**provider**: Authentication provider for the Superset service. For basic user/password authentication, the default value `db` can be used. This parameter is used internally to connect to Superset's REST API.

{% /codeInfo %}

{% codeInfo srNumber=2 %}

#### For MySQL Connection:

**username**: Specify the User to connect to MySQL. It should have enough privileges to read all the metadata.

**password**: Password to connect to MySQL.

**hostPort**: Enter the fully qualified hostname and port number for your MySQL deployment in the Host and Port field.

**Connection Options (Optional)**: Enter the details for any additional connection options that can be sent to MySQL during the connection. These details must be added as Key-Value pairs.

**Connection Arguments (Optional)**: Enter the details for any additional connection arguments such as security or protocol configs that can be sent to MySQL during the connection. These details must be added as Key-Value pairs. 
  - In case you are using Single-Sign-On (SSO) for authentication, add the `authenticator` details in the Connection Arguments as a Key-Value pair as follows: `"authenticator" : "sso_login_url"`
  - In case you authenticate with SSO using an external browser popup, then add the `authenticator` details in the Connection Arguments as a Key-Value pair as follows: `"authenticator" : "externalbrowser"`


{% /codeInfo %}

{% codeInfo srNumber=3 %}

#### For Postgres Connection:

**username**: Specify the User to connect to Postgres. It should have enough privileges to read all the metadata.

**password**: Password to connect to Postgres.

**hostPort**: Enter the fully qualified hostname and port number for your Postgres deployment in the Host and Port field.

**Connection Options (Optional)**: Enter the details for any additional connection options that can be sent to Postgres during the connection. These details must be added as Key-Value pairs.

**Connection Arguments (Optional)**: Enter the details for any additional connection arguments such as security or protocol configs that can be sent to Postgres during the connection. These details must be added as Key-Value pairs. 
  - In case you are using Single-Sign-On (SSO) for authentication, add the `authenticator` details in the Connection Arguments as a Key-Value pair as follows: `"authenticator" : "sso_login_url"`
  - In case you authenticate with SSO using an external browser popup, then add the `authenticator` details in the Connection Arguments as a Key-Value pair as follows: `"authenticator" : "externalbrowser"`

{% /codeInfo %}


#### Source Configuration - Source Config

{% codeInfo srNumber=4 %}

The `sourceConfig` is defined [here](https://github.com/open-metadata/OpenMetadata/blob/main/openmetadata-spec/src/main/resources/json/schema/metadataIngestion/dashboardServiceMetadataPipeline.json):

**dbServiceNames**: Database Service Name for the creation of lineage, if the source supports it.
**dashboardFilterPattern**, **chartFilterPattern**: Note that the they support regex as include or exclude. E.g.,
**includeTags**: Set the Include tags toggle to control whether or not to include tags as part of metadata ingestion.
**markDeletedDashboards**: Set the Mark Deleted Dashboards toggle to flag dashboards as soft-deleted if they are not present anymore in the source system.


{% /codeInfo %}


#### Sink Configuration

{% codeInfo srNumber=5 %}

To send the metadata to OpenMetadata, it needs to be specified as `type: metadata-rest`.

{% /codeInfo %}

#### Workflow Configuration

{% codeInfo srNumber=6 %}

The main property here is the `openMetadataServerConfig`, where you can define the host and security provider of your OpenMetadata installation.

For a simple, local installation using our docker containers, this looks like:

{% /codeInfo %}

{% /codeInfoContainer %}

{% codeBlock fileName="filename.yaml" %}

```yaml
source:
  type: superset
  serviceName: local_superset
  serviceConnection:
    config:
      type: Superset
```
```yaml {% srNumber=1 %}
      hostPort: http://localhost:8080
      connection:
        # For Superset API Connection
        username: admin
        password: admin
        provider: db # or provider: ldap

```
```yaml {% srNumber=2 %}
        # For MySQL Connection
        # type: Mysql
        # username: <username>
        # password: <password>
        # hostPort: <hostPort>
        # databaseSchema: superset

```
```yaml {% srNumber=3 %}
        # For Postgres Connection
        # type: Postgres
        # username: username
        # password: password
        # hostPort: localhost:5432
        # database: superset
```
```yaml {% srNumber=4 %}
  sourceConfig:
    config:
      type: DashboardMetadata
      overrideOwner: True
      # dbServiceNames:
      #   - service1
      #   - service2
      # dashboardFilterPattern:
      #   includes:
      #     - dashboard1
      #     - dashboard2
      #   excludes:
      #     - dashboard3
      #     - dashboard4
      # chartFilterPattern:
      #   includes:
      #     - chart1
      #     - chart2
      #   excludes:
      #     - chart3
      #     - chart4
```
```yaml {% srNumber=5 %}
sink:
  type: metadata-rest
  config: {}
```

```yaml {% srNumber=6 %}
workflowConfig:
  openMetadataServerConfig:
    hostPort: "http://localhost:8585/api"
    authProvider: openmetadata
    securityConfig:
      jwtToken: "{bot_jwt_token}"
```

{% /codeBlock %}

{% /codePreview %}

### Workflow Configs for Security Provider

We support different security providers. You can find their definitions [here](https://github.com/open-metadata/OpenMetadata/tree/main/openmetadata-spec/src/main/resources/json/schema/security/client).

## Openmetadata JWT Auth

- JWT tokens will allow your clients to authenticate against the OpenMetadata server. To enable JWT Tokens, you will get more details [here](/deployment/security/enable-jwt-tokens).

```yaml
workflowConfig:
  openMetadataServerConfig:
    hostPort: "http://localhost:8585/api"
    authProvider: openmetadata
    securityConfig:
      jwtToken: "{bot_jwt_token}"
```

- You can refer to the JWT Troubleshooting section [link](/deployment/security/jwt-troubleshooting) for any issues in your JWT configuration. If you need information on configuring the ingestion with other security providers in your bots, you can follow this doc [link](/deployment/security/workflow-config-auth).

### 2. Run with the CLI

First, we will need to save the YAML file. Afterward, and with all requirements installed, we can run:

```bash
metadata ingest -c <path-to-yaml>
```

Note that from connector to connector, this recipe will always be the same. By updating the YAML configuration,
you will be able to extract metadata from different sources.

