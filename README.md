# fly-log-shipper

Ship logs from fly to other providers using [NATS](https://docs.nats.io/) and [Vector](https://vector.dev/)

Here we have some vector configs, alongside a wrapper script to run it all,
that will subscribe to a log stream of your organisation's logs and ship it to various providers.

# Configuration

Create a new Fly app based on this [Dockerfile](./Dockerfile) and configure using the following secrets:

## NATS source configuration

| Secret         | Description                                                                                                      |
| -------------- |------------------------------------------------------------------------------------------------------------------|
| `ORG`          | Organisation slug (default to `personal`)                                                                        |
| `ACCESS_TOKEN` | Fly personal access token (required; set with `fly secrets set ACCESS_TOKEN=$(fly auth token)`)                  |
| `SUBJECT`      | Subject to subscribe to. See [[NATS]] below (defaults to `logs.>`)                                               |
| `QUEUE`        | Arbitrary queue name if you want to run multiple log processes for HA and avoid duplicate messages being shipped |

After generating your `fly.toml`, remember to update the internal port to match the `vector` internal port
defined in `vector-configs/vector.toml`. Not doing so will result in health checks failing on deployment.

```
[[services]]
  http_checks = []
  internal_port = 8686
```

----

Set the secrets below associated with your desired log destination

## Provider configuration

### AWS S3

| Secret                  | Description                                                                             |
| ----------------------- | --------------------------------------------------------------------------------------- |
| `AWS_ACCESS_KEY_ID`     | AWS Access key with access to the log bucket                                            |
| `AWS_SECRET_ACCESS_KEY` | AWS secret access key                                                                   |
| `AWS_BUCKET`            | AWS S3 bucket to store logs in                                                          |
| `AWS_REGION`            | Region for the bucket                                                                   |
| `S3_ENDPOINT`           | (optional) Endpoint URL for S3 compatible object stores such as Cloudflare R2 or Wasabi |

### Axiom

| Secret          | Description   |
| ----------------| --------------|
| `AXIOM_TOKEN`   | Axiom token   |
| `AXIOM_DATASET` | Axiom dataset |

### Datadog

| Secret            | Description                                   |
| ----------------- | ----------------------------------------------|
| `DATADOG_API_KEY` | API key for your Datadog account              |
| `DATADOG_SITE`    | (optional) The Datadog site. ie: datadoghq.eu |

### Honeycomb

| Secret              | Description       |
| ------------------- | ----------------- |
| `HONEYCOMB_API_KEY` | Honeycomb API key |
| `HONEYCOMB_DATASET` | Honeycomb dataset |

### Humio

| Secret           | Description                             |
| ---------------- | --------------------------------------- |
| `HUMIO_TOKEN`    | Humio token                             |
| `HUMIO_ENDPOINT` | (optional) Endpoint URL to send logs to |

### Logdna

| Secret           | Description    |
| ---------------- | -------------- |
| `LOGDNA_API_KEY` | LogDNA API key |

### Logflare

| Secret                  | Description                                             |
| ----------------------- | ------------------------------------------------------- |
| `LOGFLARE_API_KEY`      | Logflare ingest API key                                 |
| `LOGFLARE_SOURCE_TOKEN` | Logflare source token (uuid on your Logflare dashboard) |

### Logtail

| Secret          | Description        |
| --------------- | ------------------ |
| `LOGTAIL_TOKEN` | Logtail auth token |

### Loki

| Secret          | Description   |
| --------------- | ------------- |
| `LOKI_URL`      | Loki Endpoint |
| `LOKI_USERNAME` | Loki Username |
| `LOKI_PASSWORD` | Loki Password |

### New Relic

One of these is required for New Relic logs. New Relic recommend the license key be used (ref: https://docs.newrelic.com/docs/logs/enable-log-management-new-relic/enable-log-monitoring-new-relic/vector-output-sink-log-forwarding/)

| Secret                  | Description                      |
| ----------------------- | -------------------------------- |
| `NEW_RELIC_INSERT_KEY`  | (optional) New Relic Insert key  |
| `NEW_RELIC_LICENSE_KEY` | (optional) New Relic License key |

### Papertrail

| Secret                | Description         |
| --------------------- | ------------------- |
| `PAPERTRAIL_ENDPOINT` | Papertrail endpoint |

### Sematext

| Secret            | Description     |
| ----------------- | --------------- |
| `SEMATEXT_REGION` | Sematext region |
| `SEMATEXT_TOKEN`  | Sematext token  |

### Uptrace

| Secret            | Description        |
| ----------------- | ------------------ |
| `UPTRACE_API_KEY` | Uptrace API key    |
| `UPTRACE_PROJECT` | Uptrace project ID |

### EraSearch

| Secret            | Description                     |
| ----------------- | ------------------------------- |
| `ERASEARCH_URL`   | EraSearch Endpoint              |
| `ERASEARCH_AUTH`  | EraSearch User                  |
| `ERASEARCH_INDEX` | EraSearch Index you want to use |

---

# NATS

The log stream is provided through the [NATS protocol](https://docs.nats.io/nats-protocol/nats-protocol)
and is limited to subscriptions to logs in your organisations.
The NATS source takes some Fly specific environment variables to connect to the stream,
but any NATS client can connect to `fdaa::3` on port `4223` in a Fly vm,
with an organisation slug as the username and a Fly Personal Access Token as the password.

The subject schema is `logs.<app_name>.<region>.<instance_id>` and the standard
[NATS wildcards](https://docs.nats.io/nats-concepts/subjects#wildcards) can be used.
In this app, the `SUBJECT` secret can be used to set the subject and limit the scope of the logs streamed.

If you would like to run multiple vm's for high availability, the NATS endpoint supports
[subscription queues](https://docs.nats.io/nats-concepts/queue) to ensure messages are only sent to one
subscriber of the named queue. The `QUEUE` secret can be set to configure a queue name for the client.

---

# Vector

The `nats` source component sends logs to other downstream transforms and sinks in the Vector config.
This processes the log lines and sends them to various providers.
The config is generated from a shell wrapper script which uses conditionals on environment variables to
decide which Vector sinks to configure in the final config.
