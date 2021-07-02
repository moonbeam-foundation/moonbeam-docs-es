---
title: Telemetría
description: Cómo ejecutar telemetría para un nodo Parachain completo para Moonbeam Network
---

# Telemetría para un nodo completo

![Telemetry Moonbeam Banner](/images/fullnode/telemetry-banner.png)

## Introducción

Con el lanzamiento de Moonbase Alpha v6, puede activar un nodo que se conecta a Moonbase Alpha TestNet. Puede consultar esos pasos en [este tutorial](/node-operators/networks/full-node/).

Esta guía proporcionará los pasos necesarios para habilitar el servidor de telemetría para su nodo Moonbase Alpha.

!!! note
    Los pasos descritos en esta guía son para una instancia de telemetría diferente a la telemetría estándar de Polkadot habilitada de forma predeterminada (puede ejecutar nodos sin telemetría mediante el uso de la `--no-telemetry` marca). Los pasos descritos en esta guía son obligatorios solo para los nodos de clasificación.
    
## Resumen del exportador de telemetría

Moonbeam ejecutará un servidor de telemetría que recopila las métricas de Prometheus de todos los nodos de parachain de Moonbeam en la red. Ejecutar esto será de gran ayuda para nosotros durante nuestra fase de desarrollo.

El exportador de métricas puede ejecutarse como un sidecar de Kubernetes o como un binario local si está ejecutando una VM. Enviará datos a nuestros servidores, por lo que no es necesario que habilite ningún puerto de entrada para este servicio.

Estamos utilizando un servicio llamado [Gantree Node Watchdog](https://github.com/gantree-io/gantree-node-watchdog) para cargar la telemetría automáticamente. Una vez que habilita la telemetría, también puede acceder a un servidor Prometheus / Grafana desde la [aplicación Gantree](https://app.gantree.io/). Hay instrucciones detalladas en el repositorio de GitHub. Si necesita más información, aquí tiene un comienzo rápido.

Por ahora, necesitamos ejecutar dos watchdogs de nodos, uno para la parachain y otro para la cadena de retransmisión. Esto se actualizará en una versión futura. 

Para obtener ayuda, comuníquese con nuestro [servidor de Discord](https://discord.gg/FQXm74UQ7V) o con [Gantree Discord](https://discord.gg/N95McPjHZ2). 
 
## Comprobación de requisitos previos

Antes de seguir este tutorial, debe:

 1. Inicie sesión en [https://app.gantree.io](https://app.gantree.io) y cree una cuenta. Navegue a las claves de API y copie su clave de API.
 2. Solicita una clave PCK en nuestro [servidor de Discord](https://discord.gg/FQXm74UQ7V)
   
## Exportador de telemetría con Docker

Ejecutaremos dos instancias del perro guardián del nodo de Gantree usando Docker: una para la parachain y otra para la cadena de retransmisión.

### Información de configuración requerida

- GANTREE_NODE_WATCHDOG_API_KEY
- GANTREE_NODE_WATCHDOG_PROJECT_ID
- GANTREE_NODE_WATCHDOG_CLIENT_ID
- GANTREE_NODE_WATCHDOG_PCKRC
- GANTREE_NODE_WATCHDOG_METRICS_HOST

### Instrucciones

Primero, clone el repositorio del cliente de monitoreo de instancias y cree la imagen de la ventana acoplable:

```
git clone https://github.com/gantree-io/gantree-node-watchdog
cd gantree-node-watchdog
# checkout latest release
git checkout tags/$(git tag | tail -1)
docker build .  
# get the IMAGE-NAME for use below
docker images
```

A continuación, ejecutemos el contenedor de la ventana acoplable (parachain Gantree node watchdog). Tenga en cuenta que debe reemplazar los siguientes campos:

  - `IMAGE-NAME` bruja la que se buscó en el paso anterior
  - `YOUR-API-KEY` con el proporcionado por [https://app.gantree.io](https://app.gantree.io)
  - `YOUR-SERVER-NAME`
  - `YOUR-PCK-KEY` con el solicitado en nuestro servidor de Discord

```
docker run -it --network="host" \
-e GANTREE_NODE_WATCHDOG_API_KEY="YOUR-API-KEY" \
-e GANTREE_NODE_WATCHDOG_PROJECT_ID="moonbase-alpha" \
-e GANTREE_NODE_WATCHDOG_CLIENT_ID="YOUR-SERVER-NAME-parachain" \
-e GANTREE_NODE_WATCHDOG_PCKRC="YOUR-PCK-KEY" \
-e GANTREE_NODE_WATCHDOG_METRICS_HOST="http://127.0.0.1:9615" \
--name gantree_watchdog_parachain IMAGE-NAME
```

Ahora, necesitamos ejecutar el perro guardián del nodo de relé Gantree. Tenga en cuenta que debe reemplazar la misma información que en el paso anterior.

```
docker run -it --network="host" \
-e GANTREE_NODE_WATCHDOG_API_KEY="YOUR-API-KEY" \
-e GANTREE_NODE_WATCHDOG_PROJECT_ID="moonbase-alpha" \
-e GANTREE_NODE_WATCHDOG_CLIENT_ID="YOUR-SERVER-NAME-relay" \
-e GANTREE_NODE_WATCHDOG_PCKRC="YOUR-PCK-KEY" \
-e GANTREE_NODE_WATCHDOG_METRICS_HOST="http://127.0.0.1:9616" \
--name gantree_watchdog_relay IMAGE-NAME
```

Debería ver esperando el aprovisionamiento en los registros. Una vez que esté completo, puede iniciar sesión en [https://app.gantree.io](https://app.gantree.io) y seleccionar redes. Verá un `View Monitoring Dashboard` enlace a su panel personalizado de Prometheus / Grafana que puede personalizar según sus necesidades.  

Una vez que todo funcione bien, puede actualizar los comandos para que se ejecuten en modo demonio. Eliminar `-it` y agregar `-d` al comando anterior. 

## Exportador de telemetría con Systemd

Ejecutaremos dos instancias del perro guardián del nodo de Gantree: una para la parachain y otra para la cadena de relés.

### Información de configuración requerida

- GANTREE_NODE_WATCHDOG_API_KEY
- GANTREE_NODE_WATCHDOG_PROJECT_ID
- GANTREE_NODE_WATCHDOG_CLIENT_ID
- GANTREE_NODE_WATCHDOG_PCKRC
- GANTREE_NODE_WATCHDOG_METRICS_HOST

### Instrucciones

En primer lugar, tenemos que descargar el organismo de control binario de nodo Gantree desde la [página de la liberación](https://github.com/gantree-io/gantree-node-watchdog/releases), y extraerlo en una carpeta, por ejemplo, `/usr/local/bin`.

A continuación, creemos dos carpetas para los archivos de configuración:

```
mkdir -p /var/lib/gantree/parachain
mkdir -p /var/lib/gantree/relay
```

Ahora, necesitamos generar los archivos de configuración, colocar cada uno en las carpetas creadas en el paso anterior. Tenga en cuenta que debe reemplazar los siguientes campos:

  - `YOUR-API-KEY` con el proporcionado por [https://app.gantree.io](https://app.gantree.io)
  - `YOUR-SERVER-NAME`
  - `YOUR-PCK-KEY` con el solicitado en nuestro servidor de Discord
  - 
Parachain:

```
# Contents of /var/lib/gantree/parachain/.gnw_config.json
{
  "api_key": "YOUR-API-KEY",
  "project_id": "moonbase-alpha",
  "client_id": "YOUR-SERVER-NAME-parachain",
  "pckrc": "YOUR-PCK-KEY",
  "metrics_host": "http://127.0.0.1:9615"
}
```

Cadena de relés integrada:

```
# Contents of /var/lib/gantree/relay/.gnw_config.json
{
  "api_key": "YOUR-API-KEY",
  "project_id": "moonbase-alpha",
  "client_id": "YOUR-SERVER-NAME-relay",
  "pckrc": "YOUR-PCK-KEY",
  "metrics_host": "http://127.0.0.1:9616"
}
```

El siguiente paso es generar su archivo de configuración systemd.

Parachain:

```
# Contents of /etc/systemd/system/gantree-parachain.service

[Unit]
Description=Gantree Node Watchdog Parachain
After=network.target

[Service]
WorkingDirectory=/var/lib/gantree/parachain
Type=simple
Restart=always
ExecStart=/usr/local/bin/gantree_node_watchdog

[Install]
WantedBy=multi-user.target
```

Cadena de relés integrada:

```
# Contents of /etc/systemd/system/gantree-relay.service

[Unit]
Description=Gantree Node Watchdog Relay
After=network.target

[Service]
WorkingDirectory=/var/lib/gantree/relay
Type=simple
Restart=always
ExecStart=/usr/local/bin/gantree_node_watchdog

[Install]
WantedBy=multi-user.target
```

¡Estamos casi alli! Ahora, habilitemos e iniciemos los servicios de systemd, supervisemos los registros en busca de errores:

```
sudo systemctl enable gantree-parachain
sudo systemctl start gantree-parachain && journalctl -f -u gantree-parachain

sudo systemctl enable gantree-relay
sudo systemctl start gantree-relay && journalctl -f -u gantree-relay
```

Debería ver esperando el aprovisionamiento en los registros. Una vez que esté completo, puede iniciar sesión en [https://app.gantree.io](https://app.gantree.io)  y seleccionar redes. Verá un `View Monitoring Dashboard` enlace a su panel personalizado de Prometheus / Grafana que puede personalizar según sus necesidades.

## Contáctenos

Si tiene algún comentario sobre la ejecución de un nodo completo con telemetría, o cualquier otro tema relacionado con Moonbeam, no dude en comunicarse a través de nuestro [servidor de desarrollo oficial de Discord](https://discord.com/invite/PfpUATX).