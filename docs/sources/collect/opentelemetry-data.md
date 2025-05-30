---
canonical: https://grafana.com/docs/alloy/latest/collect/opentelemetry-data/
aliases:
  - ../tasks/collect-opentelemetry-data/ # /docs/alloy/latest/tasks/collect-opentelemetry-data/
description: Learn how to collect OpenTelemetry data
menuTitle: Collect OpenTelemetry data
title: Collect OpenTelemetry data and forward it to any OpenTelemetry-compatible endpoint
weight: 400
---

# Collect OpenTelemetry data and forward it to any OpenTelemetry-compatible endpoint

You can configure {{< param "PRODUCT_NAME" >}} to collect [OpenTelemetry][]-compatible data and forward it to any OpenTelemetry-compatible endpoint.

This topic describes how to:

* Configure OpenTelemetry data delivery.
* Configure batching.
* Receive OpenTelemetry data over OTLP.

## Components used in this topic

* [`otelcol.auth.basic`][otelcol.auth.basic]
* [`otelcol.exporter.otlp`][otelcol.exporter.otlp]
* [`otelcol.exporter.otlphttp`][otelcol.exporter.otlphttp]
* [`otelcol.processor.batch`][otelcol.processor.batch]
* [`otelcol.receiver.otlp`][otelcol.receiver.otlp]

## Before you begin

* Ensure that you have basic familiarity with instrumenting applications with OpenTelemetry.
* Have a set of OpenTelemetry applications ready to push telemetry data to {{< param "PRODUCT_NAME" >}}.
* Identify where {{< param "PRODUCT_NAME" >}} writes received telemetry data.
* Be familiar with the concept of [Components][] in {{< param "PRODUCT_NAME" >}}.

## Configure an OpenTelemetry Protocol exporter

Before components can receive OpenTelemetry data, you must have a component responsible for exporting the OpenTelemetry data.
An OpenTelemetry _exporter component_ is responsible for writing (exporting) OpenTelemetry data to an external system.

In this task, you use the [`otelcol.exporter.otlp`][otelcol.exporter.otlp] component to send OpenTelemetry data to a server using the OpenTelemetry Protocol (OTLP).
After an exporter component is defined, you can use other {{< param "PRODUCT_NAME" >}} components to forward data to it.

{{< admonition type="tip" >}}
Refer to the list of available [Components][] for the full list of `otelcol.exporter` components that you can use to export OpenTelemetry data.

[Components]: ../../get-started/components/
{{< /admonition >}}

To configure an `otelcol.exporter.otlp` component for exporting OpenTelemetry data using OTLP, complete the following steps:

1. Add the following `otelcol.exporter.otlp` component to your configuration file:

   ```alloy
   otelcol.exporter.otlp "<EXPORTER_LABEL>" {
     client {
       endpoint = "<HOST>:<PORT>"
     }
   }
   ```

   Replace the following:

   * _`<EXPORTER_LABEL>`_: The label for the component, such as `default`.
     The label you use must be unique across all `otelcol.exporter.otlp` components in the same configuration file.
   * _`<HOST>`_: The hostname or IP address of the server to send OTLP requests to.
   * _`<PORT>`_: The port of the server to send OTLP requests to.

1. If your server requires basic authentication, complete the following:

    1. Add the following `otelcol.auth.basic` component to your configuration file:

       ```alloy
       otelcol.auth.basic "<BASIC_AUTH_LABEL>" {
         username = "<USERNAME>"
         password = "<PASSWORD>"
       }
       ```

       Replace the following:

       * _`<BASIC_AUTH_LABEL>`_: The label for the component, such as `default`.
         The label you use must be unique across all `otelcol.auth.basic` components in the same configuration file.
       * _`<USERNAME>`_: The basic authentication username.
       * _`<PASSWORD>`_: The basic authentication password or API key.

    1. Add the following line inside of the `client` block of your `otelcol.exporter.otlp` component:

       ```alloy
       auth = otelcol.auth.basic.<BASIC_AUTH_LABEL>.handler
       ```

       Replace the following:

       * _`<BASIC_AUTH_LABEL>`_: The label for the `otelcol.auth.basic` component.

1. If you have more than one server to export metrics to, create an `otelcol.exporter.otlp` component for each additional server.

{{< admonition type="note" >}}
`otelcol.exporter.otlp` sends data using OTLP over gRPC (HTTP/2).
To send to a server using HTTP/1.1, follow the preceding steps, but use the [`otelcol.exporter.otlphttp`][otelcol.exporter.otlphttp] component instead.

[otelcol.exporter.otlphttp]: ../../reference/components/otelcol/otelcol.exporter.otlphttp/
{{< /admonition >}}

The following example demonstrates configuring `otelcol.exporter.otlp` with authentication and a component that forwards data to it:

```alloy
otelcol.exporter.otlp "default" {
  client {
    endpoint = "my-otlp-grpc-server:4317"
    auth     = otelcol.auth.basic.credentials.handler
  }
}

otelcol.auth.basic "credentials" {
  // Retrieve credentials using environment variables.

  username = sys.env("BASIC_AUTH_USER")
  password = sys.env("API_KEY")
}

otelcol.receiver.otlp "example" {
  grpc {
    endpoint = "127.0.0.1:4317"
  }

  http {
    endpoint = "127.0.0.1:4318"
  }

  output {
    metrics = [otelcol.exporter.otlp.default.input]
    logs    = [otelcol.exporter.otlp.default.input]
    traces  = [otelcol.exporter.otlp.default.input]
  }
}
```

For more information on writing OpenTelemetry data using the OpenTelemetry Protocol, refer to [`otelcol.exporter.otlp`][otelcol.exporter.otlp].

## Configure batching

Production-ready {{< param "PRODUCT_NAME" >}} configurations shouldn't send OpenTelemetry data directly to an exporter for delivery.
Instead, data is usually sent to one or more _processor components_ that perform various transformations on the data.

Ensuring data is batched is a production-readiness step to improve data compression and reduce the number of outgoing network requests to external systems.

In this task, you configure an [`otelcol.processor.batch`][otelcol.processor.batch] component to batch data before sending it to the exporter.

{{< admonition type="note" >}}
Refer to the list of available [Components][] for the full list of `otelcol.processor` components that you can use to process OpenTelemetry data.
You can chain processors by having one processor send data to another processor.

[Components]: ../../get-started/components/
{{< /admonition >}}

To configure an `otelcol.processor.batch` component, complete the following steps:

1. Follow [Configure an OpenTelemetry Protocol exporter][] to ensure received data can be written to an external system.

1. Add the following `otelcol.processor.batch` component into your configuration file:

   ```alloy
   otelcol.processor.batch "<PROCESSOR_LABEL>" {
     output {
       metrics = [otelcol.exporter.otlp.<EXPORTER_LABEL>.input]
       logs    = [otelcol.exporter.otlp.<EXPORTER_LABEL>.input]
       traces  = [otelcol.exporter.otlp.>EXPORTER_LABEL>.input]
     }
   }
   ```

   Replace the following:

   * _`<PROCESSOR_LABEL>`_: The label for the component, such as `default`.
     The label you use must be unique across all `otelcol.processor.batch` components in the same configuration file.
   * _`<EXPORTER_LABEL>`_: The label for your `otelcol.exporter.otlp` component.

   1. To disable one of the telemetry types, set the relevant type in the `output` block to the empty list, such as `metrics = []`.

   1. To send batched data to another processor, replace the components in the `output` list with the processor components to use.

The following example demonstrates configuring a sequence of `otelcol.processor` components before being exported.

```alloy
otelcol.processor.memory_limiter "default" {
  check_interval = "1s"
  limit          = "1GiB"

  output {
    metrics = [otelcol.processor.batch.default.input]
    logs    = [otelcol.processor.batch.default.input]
    traces  = [otelcol.processor.batch.default.input]
  }
}

otelcol.processor.batch "default" {
  output {
    metrics = [otelcol.exporter.otlp.default.input]
    logs    = [otelcol.exporter.otlp.default.input]
    traces  = [otelcol.exporter.otlp.default.input]
  }
}

otelcol.exporter.otlp "default" {
  client {
    endpoint = "my-otlp-grpc-server:4317"
  }
}
```

For more information on configuring OpenTelemetry data batching, refer to [`otelcol.processor.batch`][otelcol.processor.batch].

## Configure an OpenTelemetry Protocol receiver

You can configure {{< param "PRODUCT_NAME" >}} to receive OpenTelemetry metrics, logs, and traces.
An OpenTelemetry _receiver_ component is responsible for receiving OpenTelemetry data from an external system.

In this task, you use the [`otelcol.receiver.otlp`][otelcol.receiver.otlp] component to receive OpenTelemetry data over the network using the OpenTelemetry Protocol (OTLP).
You can configure a receiver component to forward received data to other {{< param "PRODUCT_NAME" >}} components.

> Refer to the list of available [Components][] for the full list of
> `otelcol.receiver` components that you can use to receive
> OpenTelemetry-compatible data.

To configure an `otelcol.receiver.otlp` component for receiving OTLP data, complete the following steps:

1. Follow [Configure an OpenTelemetry Protocol exporter][] to ensure received data can be written to an external system.

1. Optional: Follow [Configure batching][] to improve compression and reduce the total amount of network requests.

1. Add the following `otelcol.receiver.otlp` component to your configuration file.

   ```alloy
   otelcol.receiver.otlp "<LABEL>" {
     output {
       metrics = [<COMPONENT_INPUT_LIST>]
       logs    = [<COMPONENT_INPUT_LIST>]
       traces  = [<COMPONENT_INPUT_LIST>]
     }
   }
   ```

   Replace the following:

   * _`<LABEL>`_: The label for the component, such as `default`.
     The label you use must be unique across all `otelcol.receiver.otlp` components in the same configuration file.
   * _`<COMPONENT_INPUT_LIST>`_: A comma-delimited list of component inputs to forward received data to.
     For example, to send data to a batch processor component, use `otelcol.processor.batch.PROCESSOR_LABEL.input`.
     To send data directly to an exporter component, use `otelcol.exporter.otlp.EXPORTER_LABEL.input`.

   1. To allow applications to send OTLP data over gRPC on port `4317`, add the following to your `otelcol.receiver.otlp` component.

      ```alloy
      grpc {
        endpoint = "<HOST>:4317"
      }
      ```

      Replace the following:

      * _`<HOST>`_: A host to listen to traffic on. Use a narrowly scoped listen address whenever possible.
        To listen on all network interfaces, replace _`<HOST>`_ with `0.0.0.0`.

   1. To allow applications to send OTLP data over HTTP/1.1 on port `4318`, add the following to your `otelcol.receiver.otlp` component.

      ```alloy
      http {
        endpoint = "<HOST>:4318"
      }
      ```

      Replace the following:

      * _`<HOST>`_: The host to listen to traffic on. Use a narrowly scoped listen address whenever possible.
        To listen on all network interfaces, replace _`<HOST>`_ with `0.0.0.0`.

   1. To disable one of the telemetry types, set the relevant type in the `output` block to the empty list, such as `metrics = []`.

The following example demonstrates configuring `otelcol.receiver.otlp` and sending it to an exporter:

```alloy
otelcol.receiver.otlp "example" {
  grpc {
    endpoint = "127.0.0.1:4317"
  }

  http {
    endpoint = "127.0.0.1:4318"
  }

  output {
    metrics = [otelcol.processor.batch.example.input]
    logs    = [otelcol.processor.batch.example.input]
    traces  = [otelcol.processor.batch.example.input]
  }
}

otelcol.processor.batch "example" {
  output {
    metrics = [otelcol.exporter.otlp.default.input]
    logs    = [otelcol.exporter.otlp.default.input]
    traces  = [otelcol.exporter.otlp.default.input]
  }
}

otelcol.exporter.otlp "default" {
  client {
    endpoint = "my-otlp-grpc-server:4317"
  }
}
```

For more information on receiving OpenTelemetry data using the OpenTelemetry Protocol, refer to [`otelcol.receiver.otlp`][otelcol.receiver.otlp].

[OpenTelemetry]: https://opentelemetry.io
[Configure an OpenTelemetry Protocol exporter]: #configure-an-opentelemetry-protocol-exporter
[Configure batching]: #configure-batching
[otelcol.auth.basic]: ../../reference/components/otelcol/otelcol.auth.basic/
[otelcol.exporter.otlp]: ../../reference/components/otelcol/otelcol.exporter.otlp/
[otelcol.processor.batch]: ../../reference/components/otelcol/otelcol.processor.batch/
[otelcol.receiver.otlp]: ../../reference/components/otelcol/otelcol.receiver.otlp/
[otelcol.exporter.otlphttp]: ../../reference/components/otelcol/otelcol.exporter.otlphttp/
[Components]: ../../get-started/components/
