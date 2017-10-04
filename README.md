# mosquitto-kafka-integration

El presente proyecto cubre el proceso de instalación del broker MQTT Mosquitto e integración con el proyecto Apache Kafka.

## Introducción

**Mosquitto** es una implementacion open source de un broker sobre el protocolo MQTT (MQ Telemetry Transport). El proyecto de Mosquitto fue desarrollado por la Eclipse Foundation e incluye librerias escritas en C y C++, ademas de las utilidades _mosquitto_pub_ y _mosquitto_sub_ para la publicación de mensajes y la suscripción a "tópicos".

**Apache Kafka** es una proyecto open source escrito en Scala y desarrollado por la Apache Software Fundation. Kafka implementa un modelo de publicacion y subscripción para la intermediación de mensajes por medio de canales o "tópicos".

El siguiente conjunto de instrucciones tiene la finalidad de crear un punto de integración entre ambas tecnologías con el fin de apoyar proyectos que requieran de este tipo de herramientas, por ejemplo, proyectos relacionados al área de IoT (Internet of Things), Procesamiento en Streaming y Big Data.

Agradecimientos al proyecto **kafka-connect-mqtt** el cual es el núcleo de la integración entre Mosquitto y Kafka utilizado durante este proceso.

Cabe destacar que la integración explicada acontinuación es unidireccional: **Mosquitto -> Kafka**

## Pre-requisitos

* Java (1.7+)

## Versiones

* Java v1.8.0_144
* Mosquitto v1.4.14
* Apacha Kafka v0.11.0.1 Compilado en Scala 2.11
* kafka-connect-mqtt v1.1

## Instalación

En primera instancia hay que descargar e instalar el broker Mosquitto de acuerdo a las instrucciones en su sitio web oficial para cada Sistema Operativo.

Luego, para instalar las utilidades de cliente y las dependencias de Mosquitto debemos ejecutar los siguientes comandos:
```bash
apt-get update
sudo apt install mosquitto-clients
apt-get install build-essential libwrap0-dev libssl-dev libc-ares-dev uuid-dev xsltproc
```

Por otra parte, es necesario descargar Kafka desde el [sitio web oficial](https://kafka.apache.org/downloads) del proyecto de Apache. Los binarios de la version descargada incluyen Zookeeper, por lo cual no es necesario descargarlo. En caso contrario, se recomienda descargar Zookeeper desde su sitio web oficial [sitio web oficial](https://zookeeper.apache.org/releases.html#download) del proyecto.

Una vez descargado, es necesario descomprimir el archivo. De ahora en adelante nos referiremos a la ruta de descompresión de Kafka como _$KAFKA_HOME_ (ruta raíz de kafka).

Por último, necesitamos descargar y compilar la ultima versión del proyecto **kafka-connect-mqtt**, el cual provee un conector que permite capturar los mensajes recibidos en un topico de Mosquitto y publicarlos en un Topico de Kafka. Para ello necesitamos instalar Gradle, un automatizador de compilación de codigo fuente:

```bash
wget https://services.gradle.org/distributions/gradle-4.1-bin.zip
mkdir /opt/gradle
unzip -d /opt/gradle gradle-4.1-bin.zip
ls /opt/gradle/gradle-4.1
export PATH=$PATH:/opt/gradle/gradle-4.1/bin (o añadir directamente a /etc/profile)
gradle -v
```

Luego de verificar que Gradle esta correctamente instalado, descargamos/clonamos el proyecto **kafka-connect-mqtt** y lo compilamos:

```bash
git clone https://github.com/evokly/kafka-connect-mqtt.git
cd kafka-connect-mqtt
./gradlew clean jar
```

Podemos observar que en la carpeta _kafka-connect-mqtt/buid/libs_ se habrá generado la libreria que funcionara como conector entre Mosquitto y Kafka. Procedemos a copiarla a las librerias de Kafka:

```bash
cp kafka-connect-mqtt/build/libs/kafka-connect-mqtt-1.1-SNAPSHOT.jar $KAFKA_HOME/libs/
```

También es necesario descargar una libreria adicional que podemos encontrar en los binarios del proyecto **kafka-connect-mqtt**. Esta libreria tambien debe ser agregada a las librerias de Kafka:

```bash
wget https://howtoprogram.xyz/wp-content/uploads/2016/07/kafka-mqtt-bin.zip
unzip kafka-mqtt-bin.zip
cp kafka-mqtt-bin/org.eclipse.paho.client.mqttv3-1.0.2.jar $KAFKA_HOME/libs/
```

## Configuración

### Mosquitto:

En primer lugar debemos crear el usuario mosquitto en el sistema operativo (si no existe):

```bash
adduser mosquitto
```

Luego, creamos el usuario del broker que tendra los permisos para publicar mensajes y le asignamos una contraseña:

```bash
mosquitto_passwd -c /etc/mosquitto/pwfile <USUARIO_MQTT>
```
Ej:bash
```
mosquitto_passwd -c /etc/mosquitto/pwfile mosquitto
```

Se sugiere que se cree un archivo ACL el cual defina los permisos de los usuarios de Mosquitto sobre los tópicos:
bash
```
vi /etc/mosquitto/<ACL_FILE>
  user <USUARIO_MQTT>
  topic <TOPICO_DE_SERVIDOR_MQTT>
```
Ej:
```bash
vi /etc/mosquitto/aclfile
  user mosquitto
  topic mosquitto_main_topic
```

Ahora, creamos el directorio en el cual se almacenara la persistencia de Mosquitto y asignamos los permisos necesarios:

```bash
mkdir <UBICACION_PERSISTENCIA>
chown mosquitto:mosquitto <UBICACION_PERSISTENCIA> -R
```
Ej:
```bash
mkdir /var/lib/mosquitto/
chown mosquitto:mosquitto /var/lib/mosquitto/ -R
```

Es necesario que se cree/edite el archivo de configuración de Mosquitto de forma que contenga la siguiente informacion:

```bash
cp /etc/mosquitto/mosquitto.conf.example /etc/mosquitto/mosquitto.conf
vi /etc/mosquitto/mosquitto.conf
  listener 1883 <IP>
  persistence true
  persistence_location <UBICACION_PERSISTENCIA>
  persistence_file mosquitto.db
  log_dest syslog
  log_dest stdout
  log_dest topic
  log_type error
  log_type warning
  log_type notice
  log_type information
  connection_messages true
  log_timestamp true
  allow_anonymous true
  password_file /etc/mosquitto/pwfile
  acl_file /etc/mosquitto/<ACL_FILE>
```
Ej:
```bash
  listener 1883 localhost
  persistence true
  persistence_location /var/lib/mosquitto/
  persistence_file mosquitto.db
  log_dest syslog
  log_dest stdout
  log_dest topic
  log_type error
  log_type warning
  log_type notice
  log_type information
  connection_messages true
  log_timestamp true
  allow_anonymous true
  password_file /etc/mosquitto/pwfile
  acl_file /etc/mosquitto/aclfile
```

Ejecutamos el siguiente comando para actualizar los vinculos y enlaces:

```bash
/sbin/ldconfig
```

Por ultimo, eliminamos el proceso de Mosquitto en caso de que se encuentre en ejecución para que recargue la nueva configuración en el siguiente arranque del servicio:

```bash
ps -C mosquitto
sudo kill -9 MOSQUITTO_PID
```

### Kafka:

Para Kafka es necesario crear el archivo de propiedas que se usara al momento de la conneción. Ese archivo debe tener la siguiente información, la cual es requerida por el conector:

```bash
vi $KAFKA_HOME/config/mqtt.properties
  name=mqtt
  connector.class=com.evokly.kafka.connect.mqtt.MqttSourceConnector
  tasks.max=1
  kafka.topic=<TOPICO_DE_KAFKA>
  mqtt.client_id=mqtt-kafka
  mqtt.clean_session=true
  mqtt.connection_timeout=30
  mqtt.keep_alive_interval=60
  mqtt.server_uris=tcp://<IP>:1883
  mqtt.topic=<TOPICO_DE_SERVIDOR_MQTT>
  mqtt.user=<USUARIO_MQTT>
  mqtt.password=<PASSWORD>
```
Ej:
```bash
  name=mqtt
  connector.class=com.evokly.kafka.connect.mqtt.MqttSourceConnector
  tasks.max=1
  kafka.topic=hello-mqtt-kafka
  mqtt.client_id=mqtt-kafka
  mqtt.clean_session=true
  mqtt.connection_timeout=30
  mqtt.keep_alive_interval=60
  mqtt.server_uris=tcp://localhost:1883
  mqtt.topic=hello-mqtt
  mqtt.user=mosquitto
  mqtt.password=mosquitto
```

Adicionalmente, desde la varsión 0.11.0.0, Kafka incluye un parser o convertidor de valores codificados como Base64. Este convertidor sera usado por la conexión para transformar los valores recibidos de Mosquitto a formato String convencional. Para habilitar el convertidor hay que agregar la siguiente línea al final del archivo de propiedades del tipo de conexión que se usara con kafka. En nuestro caso se usara una conexión standalone, por lo que hay que modificar el archivo _$KAFKA_HOME/config/connect-standalone.properties_: 

```bash
value.converter=org.apache.kafka.connect.converters.ByteArrayConverter
```

## Ejecución

### Mosquitto:

Para iniciar el servicio de Mosquitto basta con ejecutar el siguiente comando:

```bash
mosquitto -c /etc/mosquitto/mosquitto.conf
```

### Kafka y Zookeeper:

Los servicios de Kafka y Zookeeper pueden ser iniciados con los siguientes comandos:

```bash
cd $KAFKA_HOME
./bin/zookeeper-server-start.sh config/zookeeper.properties &
./bin/kafka-server-start.sh config/server.properties &
```

### Conector Kafka-Mosquitto:

Para iniciar el conector necesitamos ejecutar el siguiente comando:

```bash
cd $KAFKA_HOME
./bin/connect-standalone.sh config/connect-standalone.properties config/mqtt.properties
```

## Suscripción y Publicación de Mensajes

Para verificar la funcionalidad del conector podemos suscribirnos al tópico de mosquitto y kafka de forma simúltanea y revisar la publicación de mensajes en ambos tópicos.

Para suscribirse al topico de Mosquitto debemos ejecutar el siguiente comando:

```bash
mosquitto_sub -h <IP> -p 1883 -v -t '<TOPICO_DE_SERVIDOR_MQTT>' -u <USUARIO_MQTT> -P <PASSWORD>
```
Ej:
```bash
mosquitto_sub -h localhost -p 1883 -v -t 'mosquitto_main_topic' -u mosquitto -P mosquitto
```

De forma similar, ejecutamos el siguiente comando para suscribirnos al topico de Kafka:

```bash
cd $KAFKA_HOME
./bin/kafka-console-consumer.sh -bootstrap-server <SERVIDOR_DE_KAFKA> --from-beginning --topic <TOPICO_DE_KAFKA>
```
Ej:
```bash
./bin/kafka-console-consumer.sh --bootstrap-server 172.31.25.244:9092 --from-beginning --topic kafka_main_topic
```

Ahora, realizamos una publicación al tópico de Mosquitto:

```bash
mosquitto_pub -h <IP> -p 1883  -t '<TOPICO_DE_SERVIDOR_MQTT>' -u <USUARIO_MQTT> -P <PASSWORD> -m '<MENSAJE>'
```
Ej:
```bash
mosquitto_pub -h localhost -p 1883  -t 'mosquitto_main_topic' -u mosquitto -P mosquitto -m 'Hello world!'
```

Podemos observar que el mensaje será publicado tanto en el tópico de Mosquitto como en el tópico de Kafka en cada una de las suscripciones.

## Información Adicional

Para aplicar offsets en la suscripción del topico de Kafka sobre la data consultada del topico de Mosquitto, se puede utilizar el viejo consumidor (Deprecado):

```bash
./bin/kafka-simple-consumer-shell.sh --broker-list <LISTA_DE_SERVIDORES_DE_KAFKA> --topic <TOPICO_DE_KAFKA> --offset <OFFSET>
```
Ej:
```bash
./bin/kafka-simple-consumer-shell.sh --broker-list localhost:9092 --topic kafka_main_topic --offset 2
```

## Referencias

[Mosquitto](https://mosquitto.org/)
[Apache Kafka](https://kafka.apache.org/)
[Apache Zookeeper](https://zookeeper.apache.org/)
[Proyecto kafka-connect-mqtt](https://github.com/evokly/kafka-connect-mqtt)

## Creditos

Mosquitto fue escrito por Roger Light roger@atchoo.org
El proyecto kafka-connect-mqtt fue deserrollado por Evokly S.A. info@evok.ly







