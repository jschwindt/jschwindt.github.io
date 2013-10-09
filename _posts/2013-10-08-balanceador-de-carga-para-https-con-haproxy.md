---
layout: post
title: "Balanceador de carga para HTTPS (SSL/TLS) con HAProxy"
category: haproxy
---

## Intro

Cuando se trata de balancear la carga HTTP me gusta utilizar HAProxy ya que es muy liviano y flexible para configurar. Pero hace unos días tuvimos la necesidad de balancear HTTPS para una aplicación de Facebook desarrollada en RoR y es por eso que me puse a investigar cómo hacerlo. En este artículo les cuento primero cómo obtener un certificado válido de SSL/TLS en forma gratuita y luego cómo configurar el HAProxy para que use dicho certificado.

## Certificado SSL/TLS gratuito

El sitio [StartSSL](http://www.startssl.com/) emite certificados SSL/TLS legítimos y gratuitos (clase 1), pero el proceso para poder utilizarlos correctamente en el servidor web o en el balanceador de carga tiene sus vueltas y me llevó un tiempo hasta que logré hacerlo funcionar correctamente.

Primero hay que registrarse en el sitio e instalar un certificado en el navegador. Ese certificado es el que nos permite luego ingresar al panel de control y gestionar todo lo necesario para obtener el certificado final. Una vez adentro, los pasos son los siguientes:

1. Ir a `Validations Wizard` y elegir `Domain Name Validation`. Este procedimiento manda un mail a alguna de las cuentas del administrador del dominio con un código de validación, que luego debe cargarse en el formulario correspondiente.

1. Una vez validado el dominio se debe generar una clave privada y luego el certificado. Para esto utilizar el `Certificates Wizard` con la opción `Web Server SSL/TLS Certificate`. Utilizar los valores default de `Keysize` y `Secure Hash Algorithm` para asegurarse la compatibilidad con navegadores viejos. Luego guardar la clave privada (como `ssl.key`, por ejemplo) y el certificado (como `ssl.crt`, por ejemplo).

1. Luego conviene eliminar el password de la clave privada para que no haga falta ingresarla cada vez que levante o reinicie el servidor web o el HAProxy. Esto se puede hacer de 2 formas: utilizando la opción `Decrypt Private Key` del `Tool Box` o mediante openssl en línea de comando:
   > openssl rsa -in ssl.key -out ssl.nokey

1. Este último paso es importante y corresponde al armado del archivo final que se va a utilizar en el servidor. Para esto es necesario descargar previamente 2 archivos desde el `Tool Box` opción `StartCom CA Certificates`: `StartCom Root CA (PEM encoded)` y `Class 1 Intermediate Server CA` y concatenarlos todo en un único archivo:
  > cat ssl.crt ssl.nokey sub.class1.server.ca.pem ca.pem >> ssl.full.crt

El archivo `ssl.full.crt` es el que utilizaremos en HAProxy según se explica en el apartado siguiente.

## HAProxy con SSL/TLS

Una gran ventaja de tener a HAProxy como punto de terminación de las conexiones SSL/TLS es que simplifica la configuración de los backend ya que la conexión hacia adentro se realiza por HTTP como siempre, y el único punto de configuración HTTPS resulta ser el proxy. Por otro lado eso también nos permite apuntar el proxy hacia diferentes aplicaciones de múltiples backends y todas compartiendo el mismo certificado.

Para utilizar HAProxy con HTTPS en necesario instalar la versión de desarrollo ya que el soporte para SSL/TLS recién comenzó a partir de la 1.5-dev12, pero ya tuvo varias revisiones y la versión actual (1.5-dev19) es muy estable y no tuve problemas en producción.

La instalación es muy sencilla y pueden ver los pasos necesarios para Debian/Ubuntu en [http://dylancopeland.com/posts/setting-up-and-configuring-haproxy](http://dylancopeland.com/posts/setting-up-and-configuring-haproxy).

En cuanto a la configuración, la directiva para habilitar el proxy de HTTPS se realiza mediante `bind` en donde se especifican los datos necesarios, por ejemplo:

```
frontend web
  bind 0.0.0.0:80
  bind 0.0.0.0:443 ssl crt /etc/haproxy/certs/ssl.full.crt
```
    
donde `ssl.full.crt` es el certificado que armamos en el apartado anterior. El resto de las directivas son las usuales para este tipo de utilización. Aquí se puede ver un ejemplo completo de la configuración: [https://gist.github.com/jschwindt/6902596](https://gist.github.com/jschwindt/6902596)

Finalmente, una vez que el HAProxy está corriendo es importante probar que toda la cadena de certificados y la compatibilidad con los distintos navegadores esté funcionando correctamente. Para eso hay varias herramientas online, por ejemplo: [http://www.sslshopper.com/ssl-checker.html](http://www.sslshopper.com/ssl-checker.html). Un problema que tuve durante las pruebas es que con Firefox aparecía un error de certificados y eso se debía a que dentro del certificado final faltaba el de la autoridad certificante intermedia, cosa que no se manifestaba con el resto de los navegadores.

## Conclusión

Dio un poco de trabajo y la información estaba un poco dispersa, pero es perfectamente posible utilizar HTTPS sin invertir dinero en comprar un certificado y utilizando las herramientas de código abierto existentes.
