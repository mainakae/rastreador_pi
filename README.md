# Escaner de redes con RbPi

Basado en [esta documentación](https://pimylifeup.com/raspberry-pi-network-scanner/), pero actualizado a lo más reciente (a fecha de escritura de este documento).

La configuración de Node-Red [obtenida aquí](https://nodered.org/docs/user-guide/runtime/securing-node-red).

## 1. Configurar la RbPi

1. Este doc esta basado en raspbian buster 32 bit
2. Usando raspberry-pi imager
3. Una vez instalado, actualiza paquetes (`sudo apt update && sudo apt upgrade -y`)
4. Usa raspi-config para:
   1. expandir fs
   2. configurar locales y timezones
   3. hostname
   4. país de la wifi
   5. nombres de interfaces predecibles

A la hora de resolver el problema de las interfaces, es posible también deshabilitar las interfaces nativas de la Rpi (bluetooth y wifi) de forma que no haya que utilizar los nombres únicos. También hay quien ha intentado hacerlo usando reglas udev, pero en mi caso no he tenido éxito con este último método.


## 2. Instalando y configurando Kismet

Los pasos serán los siguientes:
1. Cargar el repo de kismet
2. Instalar kismet
3. Añadir al usuario pi al grupo de kismet y hacer logout
4. Volver a hacer login y configurar kismet para usar nuestras interfaces
5. Configurar Kismet como servicio
6. Arrancar kismet

**1. Cargar el repositorio de Kismet**

```bash
# añadimos la clave gpg del repo
wget -O - https://www.kismetwireless.net/repos/kismet-release.gpg.key | sudo apt-key add -

# cargamos el nuevo repo en las listas de apt
echo 'deb https://www.kismetwireless.net/repos/apt/release/buster buster main' | sudo tee /etc/apt/sources.list.d/kismet.list

# actualizamos apt
sudo apt update
```

**2. Instalar Kismet**

(recordad durante la instalación, responder **YES** a la pregunta sobre usar suid helpers)

```bash
sudo apt install kismet -y
```


**3. Añadir el usuario pi al grupo kismet: `sudo usermod -aG kismet pi`**

*(ahora hacemos logout y volvemos a entrar)*

**4. Configurar las interfaces en kismet**

1. Editamos: `sudo vim /etc/Kismet/Kismet.conf`
2. Buscamos el lugar donde está lo de `source=`
```bash
# See the README for more information how to define sources; sources take the
# form of:
# source=interface:options
#
# For example to capture from a Wi-Fi interface in Linux you could specify:
# source=wlan0
#
# or to specify a custom name,
# source=wlan0:name=ath9k
#
# Sources may be defined in the config file or on the command line via the 
# '-c' option.  Sources may also be defined live via the WebUI.
#
# Kismet does not pre-define any sources, permanent sources can be added here
# or in Kismet_site.conf
source=wlan0
source=hci0
```


**5. Configurar kismet como servicio**

1. Editar `sudo systemctl edit kismet`
2. Escribir esto:
```bash
[Service]
WorkingDirectory=/home/pi
User=pi
Group=kismet
```
3. Activar el servicio: `sudo systemctl enable kismet`


## 3. Instalando node-red

1. Instalamos node-red
```bash
bash <(curl -sL https://raw.githubusercontent.com/node-red/linux-installers/master/deb/update-nodejs-and-nodered)
```
3. Lo configuramos como servicio
```bash
sudo systemctl enable nodered.service
```


## 4. Reiniciar la Rpi y jugar con node-red y kismet!

Teneis disponible el [flujo](./flows.json) para poder importarlo y trastear con el :)

Para poder hacer gráficos en grafana con los datos de influx, deberéis instalar el nodo *influxdb* en Node-Red y aseguraros de enviar datos a la bd influx.

Si no tenéis una instalación de influx, podéis lanzarla con *docker-compose* utilizando en fichero [docker-compose.yml](./docker-compose.yml) que también he incluido.

Feliz trasteo!