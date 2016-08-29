---
layout: post
title: "Arduino al rescate de una balanza"
category: electrónica
og:
  description: "Desarrollo a nuevo de una balanza de cocina que súbitamente dejó de funcionar"
  image: "/images/balanza-terminada.jpg"
---

## Motivación

Al igual que la [Pava eléctrica]({% post_url 2016-07-02-pava-digital-para-mate %}), la balanza de cocina
dejó de funcionar al poco tiempo de uso, simplemente encendía y quedaba siempre en cero. La desarmé
para ver si se podía hacer algo, pero a simple vista no había nada que pareciera estar mal, y como se trataba de
un típico [chip-on-board](https://en.wikipedia.org/wiki/Electronic_packaging#/media/File:Famicom_clone_PCB.jpg){:target="_blank" rel="noopener noreferrer"} las chances de repararla eran nulas.

De los elementos que formaban la balanza, lo único que podía ser útil era el sensor de peso o "strength gage",
así que me propuse diseñar una balanza nueva.

## El sensor de peso

Resulta que el sensor de peso de 5 Kg que traía la balanza es bastante común y existe un módulo basado en el chip
[HX711](https://cdn.sparkfun.com/datasheets/Sensors/ForceFlex/hx711_english.pdf){:target="_blank" rel="noopener noreferrer"}
que hace de amplificador de la tensión diferencial que se genera en el [puente de Wheatstone](https://es.wikipedia.org/wiki/Puente_de_Wheatstone){:target="_blank" rel="noopener noreferrer"}
y además convierten de analógico a digital. La forma de conectarlo es la siguiente:

![Cómo conectar el sensor de peso](../images/sensor_de_peso.png){:width="480px"}

También, gracias a la magia del código abierto, existe un [módulo de software](https://github.com/bogde/HX711){:target="_blank" rel="noopener noreferrer"} para Arduino para comunicarse
con el HX711 y leer el dato del peso.

## Arduino Por Mini y el display

Para este proyecto necesitaba muchas puertas de I/O: 11 salidas para manejar los 4 dígitos del display de 7
segmentos que muestra el peso, 2 para clock y datos del HX711 y una entrada leer la tecla de puesta a cero (tara).
Al mismo tiempo tenía que ser lo suficientemente pequeño para caber dentro del gabinete de la balanza. Para esto
el [Arduino Pro Mini](https://www.arduino.cc/en/Main/ArduinoBoardProMini){:target="_blank" rel="noopener noreferrer"}
resultó no sólo ser barato (no tiene la interfaz serie ni la fuente de alimentación del Arduino Uno), sino que también
tiene la separación de pines que calza exactamente con los del cuádruple display de 7 segmentos, con lo cual el montaje
quedó super compacto:

![Montaje del Arduino Pro Mini con el display](../images/montaje_display0.jpg){:width="480px"}
![Montaje del Arduino Pro Mini con el display](../images/montaje_display.jpg){:width="480px"}

En cuanto al manejo del display usé una idea sencilla que permite ahorrar componentes: tradicionalmente se suele
poner un transistor, con su resistencia en la base) por cada dígito de manera de soportar la corriente de los 7 segmentos del dígito, y 7 resistencias más, una por cada segmento, para limitar la corriente de cada led. Pero si resignamos un poco de luminosidad y encendemos de a un segmento por vez y un display por vez, sólo necesitamos 4 resistencias, una por cada display ánodo común, ya que cada salida del controlador puede manejar la corriente necesaria para un led, lo cual simplifica mucho el conexionado.

Así quedó el circuito:

[![Circuito de la balanza digital](../images/balanza-schematic.png){:width="640px"}](https://easyeda.com/jschwindt/Balanza_Digital-c0TN5nFzR){:target="_blank" rel="noopener noreferrer"}

La alimentación del circuito se realiza mediante una fuente externa de 5V, que puede provenir de un simple cargador de celular, para lo cual le puse un conector micro-usb montada al gabinete. Así quedó todo dentro de la Arduino-balanza:

![Montaje completo de la balanza](../images/montaje_completo.jpg){:width="480px"}

## Software

El soft es una clásica aplicación Arduino son su setup() y loop(), aunque decidí usar una interrupción
de timer para el multiplexado de los segmentos y dígitos. No conocía mucho del manejo de los registros
de los timers del microcontrolador, pero encontré un [buen artículo en www.instructables.com](http://www.instructables.com/id/Arduino-Timer-Interrupts/){:target="_blank" rel="noopener noreferrer"} con todo lo que necesitaba. La interrición de tiempo se genera cada 512 microsegundos y en ella lo que se hace es apagar el segmento que estaba encendido, calcular cuál es el próximo segmento y encenderlo, para lo cual hay que poner HIGH
en el ánodo del display correspondiente y LOW en el segmento. El ciclo completo dura 28 * 0.512 = 14 milisegundos, lo
que da una buena frecuencia de refresco.

El código lo publiqué en Github: [código fuente de la balanza](https://github.com/jschwindt/BalanzaDigital/blob/master/Balanza.ino){:target="_blank" rel="noopener noreferrer"}.

La balanza terminada y funcionando:

<iframe width="420" height="315" src="https://www.youtube.com/embed/HvdiXwBKR9Q" frameborder="0" allowfullscreen></iframe>
