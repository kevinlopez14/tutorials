
# KEPServerEX 6 & Grafana

En este tutorial, les mostraré la manera de crear un servidor para almacenar 
datos enviados de KEPServerEX y graficarlo en Grafana con la ayuda de distintas
tecnologias descritas a continuación. El SO utilizado en este tutorial es la 
distribución de Linux Ubuntu 20.04 (LTS) x64.




## Tecnologias necesarias

 - [Ubuntu](https://ubuntu.com/)
 - [InfluxDB](https://www.influxdata.com/)
 - [Mosquitto](https://mosquitto.org/)
 - [Telegraf](https://www.influxdata.com/time-series-platform/telegraf/)
 - [Grafana](https://grafana.com/)
 - [KEPServerEX](https://www.kepware.com/en-us/products/kepserverex/)


## InfluxDB

Instalación

```bash
  $ sudo apt-get update && sudo apt-get install influxdb
```

Accedemos a Influx
```bash
  $ influx
```
Creación de base de datos
```bash
  > create database <nombre>
```
Creación de usuario
```bash
  > CREATE USER <usuario> WITH <contraseña>
```
Le otorgamos los permisos necesarios
```bash
  > GRANT ALL PRIVILEGES TO <usuario>
```


## Mosquitto

Instalación

```bash
  $ sudo apt install mosquitto
```
Configuración

```bash
  $ cd /etc/mosquitto
```
```bash
  $ nano mosquitto.conf
```
```bash
pid_file /var/run/mosquitto.pid

persistence true
persistence_location /var/lib/mosquitto/

log_dest file /var/log/mosquitto/mosquitto.log

include_dir /etc/mosquitto/conf.d
```
Guardar Cambios y salir (Ctrl+O, Ctrl+X)

Procedemos a crear el archivo que contendrá el usuario 
y contraseña para conectarnos a través de protocolo MQTT
```bash
$ nano passwd
```
Dentro del archivo creamos los usuarios requeridos con esta estructura, uno por cada linea
```bash
<usuario>:<contraseña>
```
Guardar Cambios y salir (Ctrl+O, Ctrl+X)

A continuación debemos encriptar la contraseña, ejecutamos el siguiente comando
```bash
$ mosquitto_passwd -U passwd
```
Ya ejecutado el comando, si revisamos nuevamente el archivo veremos que la contraseña ya 
no es legible entonces lo ideal es guardarla en un lugar seguro y no olvidarla.

## Telegraf

Instalación

```bash
$ wget -qO- https://repos.influxdata.com/influxdb.key | sudo tee /etc/apt/trusted.gpg.d/influxdb.asc >/dev/null
source /etc/os-release
echo "deb https://repos.influxdata.com/${ID} ${VERSION_CODENAME} stable" | sudo tee /etc/apt/sources.list.d/influxdb.list
sudo apt-get update && sudo apt-get install telegraf
```
Configuración
```bash
  $ cd /etc/telegraf/
```
```bash
  $ nano telegraf.conf
```
Puede que veas muchas configuraciones, tienes que estudiarlas para entender su funcionalidad,
por ahora dejaremos las siguientes:
```bash
 Global Agent Configuration
[agent]
  hostname = "<identificador>"
  flush_interval = "1s"
  interval = "1s"
  metric_batch_size = 2000

# Input Plugins
[[inputs.cpu]]
    percpu = true
    totalcpu = true
    collect_cpu_time = false
    report_active = false
[[inputs.disk]]
    ignore_fs = ["tmpfs", "devtmpfs", "devfs"]
[[inputs.io]]
[[inputs.mem]]
[[inputs.net]]
[[inputs.system]]
[[inputs.swap]]
[[inputs.netstat]]
[[inputs.processes]]
[[inputs.kernel]]

# Output Plugin InfluxDB
[[outputs.influxdb]]
  database = "<base_de_datos>"
  urls = [ "http://127.0.0.1:8086" ]
  username = "<usuario_influxdb>"
  password = "<contraseña_influxdb>"

#MQTT
[[inputs.mqtt_consumer]]
  servers = ["tcp://127.0.0.1:1883"]
  topics = [
    "<topic>",
  ]
  data_format = "influx"
  connection_timeout = "30s"
  qos = 1
  max_undelivered_messages = 1000
```
Guardar Cambios y salir (Ctrl+O, Ctrl+X)


## Grafana

Instalación

```bash
  $ sudo apt-get install -y apt-transport-https
  $ sudo apt-get install -y software-properties-common wget
  $ wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
```

Agregamos el repositorio para versiones estables

```bash
$ echo "deb https://packages.grafana.com/enterprise/deb stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list
```
Instalamos
```bash
$ sudo apt-get update
$ sudo apt-get install grafana-enterprise
```
Si no funciona, y tienes localizado el archivo .deb puedes ejecutar lo siguiente

```bash
$ sudo dpkg -i grafana<edition>_<version>_amd64.deb
```
Si quieres hacer la descarga directa, este es el comando luego ejecutas el comando anterior
```bash
$ wget <.deb package url>
```
Iniciamos Grafana
```bash
$ sudo systemctl daemon-reload
$ sudo systemctl start grafana-server
$ sudo systemctl status grafana-server
```
Habilitamos el servicio para que inicie al encender o reiniciar el servidor

```bash
$ sudo systemctl enable grafana-server.service
```
## Habilitamos Firewall

Habilitar Firewall

```bash
$ sudo ufw enable
```
Ejecutamos para dar acceso a cada uno de los puertos, a los cuales se desea acceder desde
otro punto
```bash
$ sudo ufw allow <puerto>
```
Servicios y su puerto:

| Servicio             | Puerto                                                                |
| ----------------- | ------------------------------------------------------------------ |
| Grafana       | 3000 |
| Mosquitto     | 1883 |
| InfluxDB      | 8086 |

Para revisar los puertos abiertos o cerrados

```bash
$ sudo ufw status
```


## Reinicio de Servicios

Para que cada una de las configuraciones hagan efecto, tendremos que reiniciar nuestros servicios:

```bash
$ systemctl restart mosquitto
$ systemctl restart telegraf
$ systemctl restart influxdb
```

Revisamos que nuestros servicios hayan iniciado de manera correcta
```bash
$ systemctl status mosquitto
$ systemctl status telegraf
$ systemctl status influxdb
```

Si todos los servicios estan activos y funcionando de manera correcta, hemos realizado todo bien. Si ocurre un error, visita el final del documento donde hay solución a posibles errores.
## Configuración KEPServerEX

Creamos el nuevo agente dentro de KEPServerEX

![Paso 1](https://elkevo.dev/repo/Step1.png)

Nos mostrará una ventana como la siguiente, colocamos el nombre y en el tipo, verificamos que este
seleccionado MQTT Client

![Paso 2](https://elkevo.dev/repo/Step2.png)

En el siguiente paso colocaremos la configuración realizada en Telegraf, prestar mucha atención a este paso,
el topic debera ser el mismo que en nuestra configuración. 

![Paso 3](https://elkevo.dev/repo/Step3.png)

Este es el ultimo paso, colocamos el usuario y contraseña que agregamos en el paso de la instalación de Mosquitto

![Paso 4](https://elkevo.dev/repo/Step4.png)

Al finalizar, si todo lo has hecho de manera correcta, en el LOG de KEPServerEX te mostrará un mensaje asi:
```bash
Date           Time            Level           Source                               Event
5/25/2022       9:48:53 AM     Information     KEPServerEX\Runtime                  MQTT agent <agente> is connected to broker 'tcp://<ip_servidor>:1883'
```
Con esto ya solamente agregamos todos los tags que deseamos enviar a nuestro servidor.


## Posibles Errores
Mosquitto ocasionalmente tiene un error con respecto al puerto, por lo tanto para verificar esto, haremos lo siguiente:

>Creamos una nueva Screen

```bash
$ screen
```
Presionamos cualquier tecla para continuar, y tendremos la consola lista para usar. Verificamos Mosquitto:
```bash
$ mosquitto -v
```
Si te lanza un error de que no puede iniciar, porque el puerto esta siendo usado
```bash
$ lsof -i:1883
```
Nos desplegará un listado con los servicios que estan utilizando el puerto, por lo cual lo que haremos es tomar los PID y ejecutamos lo siguiente
```bash
$ kill <PID>
```
Luego de esto volvemos a ejecutar el comando
```bash
$ mosquitto -v
```
Con esto empezará a trabajar de manera correcta, dejaremos corriendo ese comando, como lo ejecutamos en una screen entonces presionamos CTRL A + D para salir de la screen
si queremos volver a esa screen donde esta ejecutando el comando de Mosquitto ejecutamos el comando
```bash
$ screen -r
```
Con esto solucionamos el problema de Mosquitto.
