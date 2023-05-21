---
title: "How I made a bruteforcer via infrared"
layout: single
excerpt: "En este artículo estaremos viendo a detalle un proyecto personal en el que he estado trabajando: Un pin bruteforcer mediante infrarrojos. Enseñare como funciona y como podeís replicarlo, no sin antes estudiar un poco el funcionamiento del infrarrojo, así como los distintos protocolos y terminologías que tiene."
show_date: true
classes: wide
toc: true
toc_label: "Contenido"
toc_icon: "fire"
header:
  teaser: https://user-images.githubusercontent.com/67548295/190915220-191d1ddf-b252-4e6c-8507-59ed0fd4382c.gif
  teaser_home_page: true
  icon: 
categories:
  - Project
tags:
  - IR
  - Bruteforce
  - Project
---
---
# Introducción
Después de tanto tiempo sin aparecer por aquí, hoy vengo con un proyecto bastante interesante y del que he aprendido mucho, se trata de un **bruteforcer** para desbloquear el **control parental** de una televisión. Antes de enseñar el script, veamos como funciona el <mark style="background: #FF5582A6;">infrarrojo</mark>. 

Al final del artículo tienes un pequeño glosario donde podrás consultar la terminología que uso.

## ¿Qúe es el infrarrojo? ¿Cómo funciona?
Si nos fijamos en la mayoría de los mandos a distancia de las televisiones, podremos notar que tienen una **pequeña bombilla** en la parte superior (no todos)

<p align="center">
  <img src="https://user-images.githubusercontent.com/67548295/190917156-b24d5bbb-5fae-4b57-93f7-9d36f02a9fdf.jpeg" />
</p>

Esta bombilla es el <mark style="background: #FF5582A6;">emisor infrarrojo</mark>, la cual emite pulsos de luz con una **longitud de onda** de entre <mark style="background: #ADCCFFA6;">700 nm y 1 mm</mark>, rango el cual está fuera del **espectro visible**.

<p align="center">
  <img src="https://user-images.githubusercontent.com/67548295/190920967-5a3b8547-591b-40fc-90af-6d806e346c38.png" />
</p>

La información se emite apagando y encendiendo el LED miles de veces por segundo, la velocidad a la sucede esto se denomina como **frecuencia del portador** (del inglés **carrier frequency**). Establecer esta frecuencia para la transmisión de datos trae consigo la ventaja de poder envíar y recibir datos en entornos "ruidosos" (con interferencias por otra luz del entorno).

La información se almacena en el "portador" (mando a distancia) en **formato binario** y esta se transmite. La forma más común de transmisión es la que se conoce [Pulse-Width Modulation](https://en.wikipedia.org/wiki/Pulse-width_modulation){:target="\_blank"}{:rel="noopener nofollow"} o **Modulación del ancho del pulso**, esta técnica juega con los periodos de **encendido** y **apagado**. Cada fabricante elige que números almacenar y enviar y el significado de cada uno de ellos. Debido a que siempre hay algo de "**_ruido_**" en el entorno (Puede ser cualquier cosa, la luz del sol u otras luces brillantes) algunos fabricantes también agregan **información adicional** en la transmisión para asegurar que lleguen al equipo correctamente, como enviar los números 2 veces 

Resumiendo, la transmisión de [IR](#glosario) se lleva a cabo mayoritariamente variando los tiempos de encendido y apagado del emisor para representar números binarios siguiendo un patrón establecido.

## NEC protocol
Profundicemos un poco más para ver uno de los protocolos que hace uso del [PWM](#glosario), el <mark style="background: #FF5582A6;">protocolo NEC</mark>.

En este protocolo, las **ráfagas** de luz infrarroja se transmiten en una frecuencia de <mark style="background: #BBFABBA6;">38KHz</mark> y cada pulso mide **562.5 [µs](#glosario)** (Lo que corresponde a 21 ciclos de onda en esa frecuencia)
un **1** lógico tarda **2.25ms** en trasmitirse y un **0** lógico tarda la mitad, **1.125ms**.

<p align="center">
   <img src="https://user-images.githubusercontent.com/67548295/190926043-29bba8a0-6fde-4342-b997-238d2ec38102.png"/>
</p>

En la imagen nos podemos dar cuenta de donde vienen esos tiempos de transmisión. El **1** lógico está compuesto por una ráfaga de **562.5 µs** y seguido por espacio de **1.6875ms**, resultando en <mark style="background: #BBFABBA6;">2.25ms</mark> de transmisión en total, y un **0** lógico está compuesto también por una ráfaga de **562.5 µs** y seguido por un espacio de la misma longitud, resultando en un tiempo de transmisión total de <mark style="background: #BBFABBA6;">1.125ms</mark>.

Cuando se pulsa una tecla del mando a distancia, el mensaje que se transmite es el siguiente:

<p align="center">
	<img src="https://user-images.githubusercontent.com/67548295/190927250-13fb879e-0a99-4b65-af42-8f999cdb28d8.png"/>
</p>

El mensaje consiste en los siguiente:

* Una ráfaga inicial de **9ms** con la que empieza la transmisión
* Una espacio de **4.5ms**
* Una dirección para el recibidor de 8 bits
* 8 bits **invertidos** de la dirección
* Un comando en 8 bits
* 8 bits **invertidos** del comando
* Una ráfaga de **562.5 µs** para indicar el final del mensaje

> Nota: En los 4 bytes enviados siempre se envía primero el [LSB](#glosario)

Probablemente te preguntes (o no) que pasa si dejamos un botón del mando a distancia presionado. Lo que pasa es que se envía un **código de repetición** que indica que se va a repetir el mensaje anterior. Este código normalmente se envía **40ms** después de la ráfaga de **562.5 µs** que indica el final del mensaje.

El código de repetición se ve así:

<p align="center">
	<img src="https://user-images.githubusercontent.com/67548295/190929478-d6a0598e-57ae-46f2-b58c-b5b6e61a308a.png"/>
</p>

Este consiste en lo siguiente:
* Una ráfaga de pulsos de **9ms***
* Un espacio de 2.25 **ms**
* Una ráfaga de **562.5 µs** que indica el final del código de repetición

## Pronto Hex
Algo también muy importante que debemos ver es el formato **Pronto Hex**.
A continuación podemos ver una emisión infrarroja en este formato:
```txt
0000 0070 0003 0002 0006 0002 0004 0002 0004  
0006 0006 0003 0003 000С
```

Como podemos ver, está dividido por <mark style="background: #BBFABBA6;">bloques en hexadecimal</mark> que llamaremos **palabras**. El significado de cada una de estas es el siguiente:
* **Offset 0 (0000)**: ID del formato, mide un bloque y  corresponde al formato con el que estaremos trabajando: el formato [raw](#glosario).
* **Offset 1 (0070)**: Divisor del [Carrier frequency](#glosario), mide un bloque. Para obtener la <mark style="background: #FFF3A3A6;">frecuencia en KHz</mark> se utiliza la siguiente formula: `1000000 / (N * 0.241246)` donde N es el <u>valor decimal</u> del bloque. En este caso la frecuencia es de **37037 KHz**
* **Offset 2 (0003)**: Número de pares de ráfagas en la primera secuencia. Cada par de ráfagas indica los tiempos de apagado y encendido de la secuencia. Mide un bloque.
* **Offset 3 (0002)**: Número de pares de ráfagas en la secuencia de repetición. Mide un bloque.
* **Offset 4 (0006 0002 0004 0002  
0004 0006)**: Secuencia a emitir. Mide 2 * (offset 2), en este caso **6** bloques. 
* **Offset 10 (4 + 2*(offset 2)) (0006 0003 0003 000C)**: Secuencia de repetición. Mide 2 * (offset3), entes caso **4** bloques 

Un <mark style="background: #BBFABBA6;">par de ráfagas</mark> , si no ha quedado claro es lo siguiente: Pongamos de ejemplo el par: `0006 0002`:

| Offset | Tamaño | Estado del LED | Descripción                                                                                  | Muestra |
| ------ | ------ | -------------- | -------------------------------------------------------------------------------------------- | ------- |
| 0      | 1      | Parpadeando    | Cantidad de [periodos](#glosario) en los que el LED parpadea acorde al **carrier frequency** | `0006` |
| 1      | 1      | Apagado        | Cantidad de periodos en los que el LED está apagado                                          | `0002` |        |

---

# Pin bruteforcer
Al comenzar a investigar en este proyecto, me dí cuenta de que lo primero que tenía que hacer era encontrar los patrones de emisión en hexadecimal correspondientes a mi televisión. En mi larga búsqueda dí con esta mina de oro:

* [IRDB](https://github.com/probonopd/irdb/){:target="\_blank"}{:rel="noopener nofollow"}

Se trata de una de las mayores bases de datos de códigos infrarrojos. Contiene código casi para cualquier dispositivo que haga uso de esta tecnología. Busqúe los códigos correspondientes al fabricante de mi TV y estaban, el único inconveniente que tenía era que estos estaban en un formato que no había visto:
<p align="center">
	<img src="https://user-images.githubusercontent.com/67548295/191079675-30ad719f-ce37-42c0-9022-29941362ab24.png"/>
</p>

¿**Device**, **Subdevice** y **Function**? No sabía lo que era, pero había algo que me preocupaba más, ¿Cómo iba a transmitir esos códigos? Había visto que se podía usar una placa [Arduino](https://www.arduino.cc/){:target="\_blank"}{:rel="noopener nofollow"} con emisores infrarrojos, pero nunca había tenido esa placa y no sabía como funcionaba.

Más tarde me dí cuenta de que mi telefono móvil cuenta con un **emisor infrarrojo** y buscando maneras para poder enviar mis propias secuencias, veo que hay una aplicación que emula una terminal unix llamada [Termux](https://f-droid.org/packages/com.termux/){:target="\_blank"}{:rel="noopener nofollow"} que junto con su [API](https://f-droid.org/en/packages/com.termux.api/) permiten lo que buscaba. 

Bien, para montar tu entorno para enviar tus secuencias desde tu teléfono (tiene que tener infrarrojo) deberás seguir los siguientes pasos:

###### 1º Paso | Instalar Termux desde Fdroid
Sé que esta app está en la [Play Store](https://play.google.com/store/apps){:target="\_blank"}{:rel="noopener nofollow"} pero debido a un cambio de políticas que hubo ya no está siendo actualizada en esta plataforma. Podemos descargar la última versión desde [aquí](https://f-droid.org/en/packages/com.termux/){:target="\_blank"}{:rel="noopener nofollow"}

###### 2º Paso | Actualizar repositorios e instalar algunos paquetes
Una vez descargada e instalada la app, entraremos y ejcutaremos los siguientes comandos:
```sh
apt update && pkg upgrade -y

pkg install nc termux-api
```

###### 3º Paso | Instalar Termux-api desde Fdroid
Ahora tendremos que instalar la propia aplicación de Termux-api la cual se puede descargar desde [Fdroid](https://f-droid.org/en/packages/com.termux.api/){:target="\_blank"}{:rel="noopener nofollow"}

###### 4º Paso | Ejecutar el script que nos interesa
Una vez completados todos los pasos, ya tenemos listo nuestro entorno.
El programa que vamos a estar utilizando es `termux-infrared-transmit`.
Si lo ejecutamos desde **Termux** con el parámetro `-h` veremos las opciones que se nos requieren:

<p align="center">
	<img src="https://user-images.githubusercontent.com/67548295/191097468-8009d97a-cfd0-4a23-8571-d47c2b6b90eb.jpg"/>
</p>

Por un lado tendríamos que especificar el **carrier frequency** con el parámetro `-f` y luego los **intervalos** de encendido y apagado separados por comas. Estos dos valores los podemos sacar del formato **Ponto HEX**, pero por ahora lo que tenemos es los valores de **device**, **subdevice** y **function**. ¿Cómo convertimos esos valores a **Pronto HEX**?

### IrScrutinizer

Hay un programa maravilloso llamado [IrScrutinizer](http://www.hifi-remote.com/wiki/index.php?title=IrScrutinizer_Guide){:target="\_blank"}{:rel="noopener nofollow"} que automatiza esta tarea y muchísimas más. Ahorra el dolor de cabeza de tener que hacerlo de forma manual. Podremos instalarlo desde su [repositorio en GitHub](https://github.com/bengtmartensson/IrScrutinizer/releases/tag/Version-2.3.1){:target="\_blank"}{:rel="noopener nofollow"}, si manejas un sistema Linux, descarga el [AppImage](https://github.com/bengtmartensson/IrScrutinizer/releases/download/Version-2.3.1/IrScrutinizer-2.3.1-x86_64.AppImage){:target="\_blank"}{:rel="noopener nofollow"} o si manejas un sistema Windows pues descarga el [EXE](https://github.com/bengtmartensson/IrScrutinizer/releases/download/Version-2.3.1/IrScrutinizer-2.3.1.exe){:target="\_blank"}{:rel="noopener nofollow"}

Una vez instalado, para generar códigos IR en formato **Pronto HEX** es muy sencillo:

<p align="center">
	<img src="https://user-images.githubusercontent.com/67548295/191778673-04a65780-677d-4672-b0dc-bb74bc62e432.png"/>
</p>

Lo primero es ir a la pestaña **Render**. Luego tendremos que rellenar los campos con los valores de **Device**, **Subdevice** y **Function**. Probemos a generar el código para el botón de apagado/encendido:

| Acción | Protocolo | Device | Subdevice | Function |
| ------ | --------- | ------ | --------- | -------- |
| POWER  | NEC1      | 4      | -1        | 8         |

<p align="center">
	<img src="https://user-images.githubusercontent.com/67548295/191786733-1ca432df-29ae-4213-a35d-f2a4bc6f34ad.png"/>
</p>

Una vez hecho eso, hacemos clic en el botón **Open last export** y podremos ver el código **Pronto HEX**

<p align="center">
	<img src="https://user-images.githubusercontent.com/67548295/191788743-ee74a9b5-e394-4640-b1da-c680bbcdfcff.png"/>
</p>

Para hacer el **pin bruteforcer** necesitaríamos el código Pronto HEX de todos los números, pero por suerte alguien ya nos ha proporcionado esta lista en un [foro](https://www.remotecentral.com/cgi-bin/mboard/rc-discrete/thread.cgi?7244){:target="\_blank"}{:rel="noopener nofollow"}

### Creación del script
Una vez obtuve todos los códigos **Pronto HEX**, hice un script con para convertirlos (con las **fórmulas** que vimos anteriormente) a periodos de encendido y apagado para que **Termux** pudiese interpretarlos.

```python
#!/usr/bin/python3

power = "0000 006D 0022 0002 0155 00AA 0015 0015 0015 0015 0015 0015 0015 0015 0015 0015 0015 0015 0015 0015 0015 0015 0015 0015 0015 0015 0015 0015 0015 0015 0015 0015 0015 0015 0015 0015 0015 0015 0015 0040 0015 0015 0015 0015 0015 0015 0015 0015 0015 0015 0015 0015 0015 0015 0015 0040 0015 0040 0015 0015 0015 0040 0015 0040 0015 0040 0015 0015 0015 0040 0015 05ED 0155 0055 0015 0E47"

def command_generator(keys):
    keys = keys.split(" ")
    freq = round(1000000 / (int(keys[1], 16) * .241246))

    milisecs = []
    for hex_value in keys:
        milisecs.append(round(1000000 * int(hex_value, 16) / freq))

    to_str = ','.join(str(x) for x in milisecs[4:])

    return f"termux-infrared-transmit -f {freq} {to_str};"

print(command_generator(power))
```

Si ejecutamos este script nos generaría un comando listo para usar, algo tal que así:

```sh
termux-infrared-transmit -f 38029 8967,4470, etc
```

Ejecutamos este script en nuestro PC y envíamos el ouput hacia el teléfono con el comando `nc` :
```bash
python3 command_generator.py | nc 192.168.1.10 4444
```
Reemplazamos la la IP por la **IP local** de nuestro teléfono (Es necesario que el equipo host y el teléfono estén conectados a la **misma red**, la IP local se puede consultar con el comando `ifconfig`). Antes de ejecutar este comando, tendremos que estar en escucha con **Termux** con este comando:

```bash
nc -lnp 4444 -k | sh
```
Ese comando lo que hace es quedarse en escucha por el puerto **4444** y que todo lo que le enviemos como data lo ejecute con **sh**

---

Bien, el<mark style="background: #BBFABBA6;"> control parental</mark> de mi TV me dejaba insertar **3** contraseñas incorrectas, después de eso se cerraba el prompt y había que darle al botón **OK** para hacerlo aparecer de nuevo. A partir de eso, hice el bruteforcer que teneís en mi [GitHub](https://github.com/z3rObyte/LG-infrared-pin-bruteforcer){:target="\_blank"}{:rel="noopener nofollow"}

Aquí teneis un video de la herramienta en acción:


<video src="https://user-images.githubusercontent.com/67548295/191803526-fb5b41d0-2c48-4267-bef2-45ad20ddd4cc.mp4" controls="controls" style="max-width: 730px; max-height: 730px;" >
</video>

# Conclusión
Este ha sido un proyecto en el que he aprendido mucho. En mi opinión, si sabes hacer estas cosas, el pin del control parental no sirve para nada ya que ni siquiera tiene un cooldown para evitar esto.
Me ha gustado mucho tanto investigar en este campo como escribir este artículo, es por eso que posiblemente traiga más cosas así por el blog.







# Glosario
* **IR**: Infrarrojo (Del inglés infrared)
* **Carrier frequency**: Frecuencia de una <mark style="background: #FF5582A6;">carrier wave</mark> o <mark style="background: #FF5582A6;">onda portadora</mark> 
* **Carrier wave**:  Onda que se modula (modifica) con una señal portadora de información con el fin de transmitirla.
* **PWM**: Acrónimo de [Pulse-Width Modulation](https://en.wikipedia.org/wiki/Pulse-width_modulation){:target="\_blank"}{:rel="noopener nofollow"} o Modulación del ancho del pulso
* **Medida µs**: Microsegundos, 1 milisegundo = 1000 microsegundos
* **LSB**: Bit menos significativo (del inglés, Least Significant Bit) posición del bit en un entero binario que da el valor de las unidades.
* **Periodo**: Acorde a [Wikipedia](https://www.wikipedia.org/){:target="\_blank"}{:rel="noopener nofollow"}, el periodo es la duración del tiempo de un ciclo en un evento que se repite
* **raw**: Datos sin procesar





