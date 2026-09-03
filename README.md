# OpenSearch Docker Compose - Multi-Node Cluster

A Docker Compose setup for a two-node OpenSearch cluster with TLS, the OpenSearch Security plugin, automatically generated credentials configuration, and OpenSearch Dashboards.

## Features

- Two-node OpenSearch cluster
- Automatic Root CA and service certificate generation
- Automatic `internal_users.yml` generation from `.env` passwords
- Automatic bcrypt password hashing with OpenSearch's bundled `hash.sh`
- Automatic security-index update with `securityadmin.sh`
- OpenSearch Dashboards using a dedicated service account
- Persistent OpenSearch data volumes

## Quick Start

1. Copy the example environment file:

   ```bash
   cp example.env .env
   ```

2. Edit `.env` and replace at least these passwords:

   ```text
   OPENSEARCH_INITIAL_ADMIN_PASSWORD=...
   OPENSEARCH_DASHBOARDS_INTERNAL_PASSWORD=...
   ```

3. Start the cluster:

   ```bash
   docker compose up -d
   ```

4. Check startup status:

   ```bash
   docker compose ps -a
   ```

   The expected steady state is:

   - `opensearch-setup`: exited successfully
   - `opensearch-security-setup`: exited successfully
   - `opensearch-node1`: running/healthy
   - `opensearch-node2`: running/healthy
   - `opensearch-dashboards`: running

5. Open OpenSearch Dashboards at `http://localhost:5601`.

   Sign in with:

   ```text
   username: admin
   password: value of OPENSEARCH_INITIAL_ADMIN_PASSWORD
   ```

6. Test the OpenSearch API:

   ```bash
   curl -k \
     -u "admin:<value-of-OPENSEARCH_INITIAL_ADMIN_PASSWORD>" \
     https://localhost:9200
   ```

## Services

### `opensearch-setup`

A one-shot initialization service that:

1. Generates the Root CA and TLS certificates if they do not already exist.
2. Derives the actual certificate DNs and updates the two OpenSearch node configuration files.
3. Copies the version-matched `opensearch-security` configuration from the OpenSearch image into the `security-config` volume.
4. Hashes `OPENSEARCH_INITIAL_ADMIN_PASSWORD` using OpenSearch's bundled `hash.sh`.
5. Hashes `OPENSEARCH_DASHBOARDS_INTERNAL_PASSWORD` using the same tool.
6. Generates `internal_users.yml` containing:
   - the `admin` user
   - the configured Dashboards service account

The user does not need to manually generate bcrypt hashes or edit `internal_users.yml`.

### `opensearch-node1` and `opensearch-node2`

The two OpenSearch nodes mount:

- generated TLS certificates from `certs`
- generated Security-plugin configuration from `security-config`
- persistent data volumes

The OpenSearch demo configuration installer is explicitly disabled because this stack supplies its own TLS and security configuration.

### `opensearch-security-setup`

A second one-shot service that starts after both OpenSearch nodes are healthy. It applies the generated `internal_users.yml` with the admin certificate and OpenSearch's bundled `securityadmin.sh`.

This makes password changes in `.env` take effect when the Compose application is recreated with:

```bash
docker compose up -d --force-recreate
```

**Important:** This stack treats the generated `internal_users.yml` as the source of truth for internal users. `opensearch-security-setup` replaces the `internalusers` portion of the Security index with the generated file. Internal users created manually through the Security REST API or Dashboards will therefore be removed the next time this setup service reapplies the generated file.

### `opensearch-dashboards`

Dashboards starts only after `opensearch-security-setup` completes successfully.

`OPENSEARCH_DASHBOARDS_INTERNAL_USERNAME` and `OPENSEARCH_DASHBOARDS_INTERNAL_PASSWORD` are the **server-to-server service-account credentials** used by Dashboards to communicate with OpenSearch. They are not the normal web UI login credentials.

The default service account name is `kibanaserver` and is automatically assigned the `kibana_server` security role.

## Environment Variables

| Variable | Example/default | Description |
|---|---|---|
| `COMPOSE_PROJECT_NAME` | `opensearch-cluster` | Compose project name |
| `OPENSEARCH_VERSION` | `latest` | OpenSearch image tag |
| `OPENSEARCH_DASHBOARDS_VERSION` | `latest` | Dashboards image tag |
| `OPENSEARCH_CLUSTER_NAME` | `opensearch-cluster` | OpenSearch cluster name |
| `OPENSEARCH_INITIAL_ADMIN_PASSWORD` | required | Password generated into the `admin` internal user |
| `OPENSEARCH_DASHBOARDS_INTERNAL_USERNAME` | `kibanaserver` | Dashboards service-account username |
| `OPENSEARCH_DASHBOARDS_INTERNAL_PASSWORD` | required | Dashboards service-account password |
| `OPENSEARCH_JAVA_OPTS` | `-Xms2g -Xmx2g` | OpenSearch JVM options |
| `CERT_ORGANIZATION` | `OpenSearch` | Certificate organization |
| `CERT_COUNTRY` | `US` | Certificate country |
| `CERT_STATE` | `CA` | Certificate state/province |
| `CERT_LOCALITY` | `San Francisco` | Certificate locality |

See `example.env` for the complete list.

## Generated Security Configuration

The `security-config` named volume contains the generated OpenSearch Security configuration. The setup process starts with the configuration shipped in the selected OpenSearch image, so files such as `roles.yml`, `roles_mapping.yml`, `config.yml`, and `action_groups.yml` stay aligned with `OPENSEARCH_VERSION`.

`internal_users.yml` is then replaced with a generated file containing only the Compose-managed users.

For the default configuration it is equivalent to:

```yaml
---
_meta:
  type: "internalusers"
  config_version: 2

admin:
  hash: "<generated bcrypt hash>"
  reserved: true
  backend_roles:
    - "admin"
  description: "Admin user"

kibanaserver:
  hash: "<generated bcrypt hash>"
  reserved: true
  opendistro_security_roles:
    - "kibana_server"
  description: "OpenSearch Dashboards server user"
```

## Certificate Configuration

Certificates are stored in the `certs` named volume. The setup container derives the configured DNs directly from the certificates, which prevents an existing certificate volume from becoming inconsistent with the node YAML files.

To change certificate subject values, remove only the certificate volume and recreate the application. Do **not** use `docker compose down -v` unless you intentionally want to delete the OpenSearch data volumes too.

A safe sequence is:

```bash
docker compose down
docker volume ls --filter label=com.docker.compose.volume=certs
docker volume rm <the-certs-volume-name>
docker compose up -d
```

Use the Compose labels shown by `docker volume inspect <volume>` to confirm the volume belongs to this project before removing it.

## Password Changes

Edit `.env`, then recreate the setup/security/Dashboards containers:

```bash
docker compose up -d --force-recreate
```

The setup container regenerates `internal_users.yml`, and `opensearch-security-setup` applies it to the existing Security index.

## Troubleshooting

### Setup

```bash
docker compose logs opensearch-setup
```

### Security configuration

```bash
docker compose logs opensearch-security-setup
```

### OpenSearch nodes

```bash
docker compose logs opensearch-node1 opensearch-node2
```

### Dashboards

```bash
docker compose logs opensearch-dashboards
```

### Verify the Dashboards service-account environment

```bash
docker compose exec opensearch-dashboards env | grep '^OPENSEARCH_'
```

### Verify admin authentication

```bash
curl -k \
  -u "admin:<value-of-OPENSEARCH_INITIAL_ADMIN_PASSWORD>" \
  https://localhost:9200/_cluster/health?pretty
```

## Stopping the Cluster

Preserve all volumes:

```bash
docker compose down
```

Delete all generated certificates, generated security configuration, and OpenSearch data:

```bash
docker compose down -v
```

The second command is destructive and removes the data volumes.

## License

See [LICENSE](LICENSE).
