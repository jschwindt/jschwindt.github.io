---
layout: post
title: "Pava digital para mate"
category: electrónica
og:
  description: "Reemplazo del termostato mecánico por un microcontrolador ATtiny85 a una pava eléctrica"
  image: "/image/pava-mate-digital.png"
---

## Motivación

Compré una pava eléctrica con control de temperatura para el agua del mate, pero al poco tiempo el
termostato de 85ºC se rompió y no hubo forma de conseguir el repuesto. La molestia de estar siempre atento
a que el agua no hierva hizo que el problema se convirtiera rápidamente en una oportunidad para jugar con
Arduino en un proyecto concreto de "digitalizar" la pava.

## El sensor de temperatura

La primera idea fue usar el [LM335](http://www.ti.com/lit/ds/symlink/lm235.pdf){:target="_blank"} ya que tenía uno a mano. Este
sensor entrega 10mV por ºK, es decir que para el rango de 10 a 90ºC del agua, la tensión que entrega es de
2,83 a 3,63V, es decir menos de 1V de variación, y como la idea es utilizar el ADC del Arduino, que mide de 0 a 5V, se
desperdiciaría el 80% del rango de conversión. Otro problema es que al tener un encapsulado plástico, el
contacto térmico y la inercia térmica complicaban el control.

Finalmente me sugirieron utilizar un NTC 100K a 25ºC con encapsulado en vidrio, que se usa en las impresoras 3D
y es muy barato. Con esto es muy fácil regular el rango de tensión cambiando la resistencia del divisor, la que
quedó en 22K. La respuesta del divisor de tensión no es lineal ([ver tabla de resistencia del NTC](/images/tabla-ntc-100k.png){:target="_blank"}), pero para el uso como termostato eso no es importante.

Así quedó el termistor dentro del espacio del viejo termostato:

![Así quedó el termistor en el lugar del termostato](../images/termistor.jpg){:width="300px"}

## El circuito y el armado con el ATtiny85

El circuito definitivo es muy sencillo ya que el microcontrolador no necesita ningún componente externo para funcionar:
![Circuito de la pava para mate](../images/pava-mate-digital.png){:width="480px"}

Esta es la plaquetita terminada:
![Plaqueta terminada](../images/placa.jpg){:width="480px"}

La fuente de alimentación tenía que ser pequeña para que entrara en la base de la pava eléctrica, así que lo más fácil
fue usar un cargador de celular sin su encapsulado plástico.

Y todo montado en la base de la pava:

![Montaje en la base de la pava eléctrica](../images/montaje.jpg){:width="480px"}

## Software

El soft es muy sencillo, pero como hay que tener en cuenta la inercia térmica de la pava y del sensor, fue necesario
hacer una pequeña máquina de estado. El funcionamiento es el siguiente: primero calienta el agua hasta alcanzar aproximadamente
los 60ºC. A partir de ahí comienzan ciclos de 10 segundos de espera (para que se estabilice la medición de temperatura), y
7 segundos de calentamiento. Cuando se llega a los 85ºC buscados, se apaga la resistencia y se pasa a un estado de espera hasta que la temperatura pasa por debajo de los 80ºC, donde vuelve a empezar el ciclo.

El LED se utiliza para saber en qué parte del ciclo se encuentra: si está encendido en forma permanente significa que el
agua se encuentra lista para preparar el mate. Si estamos en el proceso de calentamiento, el LED titila con una velocidad
variable y se acelera a medida que nos acercamos a los 85ºC. El código lo publiqué en Github: [código fuente de la pava](https://github.com/jschwindt/PavaMateDigital/blob/master/PavaMate.ino){:target="_blank"}.

Durante el desarrollo usé un Arduino Uno ya que de esa forma podía usar la interfaz serie para enviar mensajes de debug y de esa forma acelerar el proceso. Luego, cuando llegó el momento de programar definitivamente el ATtiny85, seguí las instrucciones
de estos tutoriales [Video: How to make a DIY ATtiny Programmer](https://www.youtube.com/watch?v=BexXvxmOGN8){:target="_blank"} y [Programming an ATtiny w/ Arduino 1.6](http://highlowtech.org/?p=1695){:target="_blank"}.

La pava "electrónica" terminada:

<iframe width="420" height="315" src="https://www.youtube.com/embed/4MAwimaxqno" frameborder="0" allowfullscreen></iframe>
