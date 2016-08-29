---
layout: post
title: "Arduino al rescate de una balanza"
category: electrónica
og:
  description: "Desarrollo a nuevo de una balanza de cocina que súbitamente dejó de funcionar"
  image: "/images/promini.jpg"
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
resultó no sólo ser barato (no tiene la interfaz serie ni fuente de alimentación del Arduino Uno), sino que también
tiene la separación de pines que calza exactamente con los del cuádruple display de 7 segmentos, con lo cual el montaje
quedó super compacto:

![Montaje del Arduino Pro Mini con el display](../images/montaje_display0.jpg){:width="480px"}
![Montaje del Arduino Pro Mini con el display](../images/montaje_display.jpg){:width="480px"}

En cuanto al manejo del display usé una idea sencilla que permite ahorrar componentes. Tradicionalmente se suele
poner un transistor por cada dígito de manera de soportar la corriente de los 7 segmentos del dígito, ya que cada salida
del microcontrolador no soporta más de 40mA. Además es necesario poner una resistencia por cada segmento (total: 11 resistencias y 4 transistores). Pero si resignamos un poco de luminosidad y encendemos un segmento por vez y un
display por vez, sólo necesitamos 4 resistencias, una por cada display:

Así quedó el circuito:

[![Circuito de la balanza digital](../images/balanza-schematic.png){:width="640px"}](https://easyeda.com/jschwindt/Balanza_Digital-c0TN5nFzR){:target="_blank" rel="noopener noreferrer"}


Montaje completo de la Arduino-balanza:

![Montaje completo de la balanza](../images/montaje_completo.jpg){:width="480px"}

La balanza terminada y funcionando:

<iframe width="420" height="315" src="https://www.youtube.com/watch?v=HvdiXwBKR9Q" frameborder="0" allowfullscreen></iframe>
