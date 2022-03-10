# Examen Semana 1 

## Instrucción 

El propósito del Examen 01 es el de aplicar los conocimientos de certificados digitales, dentro de la configuración de servicios/servidores en Linux.

En este sentido se debe prepara una maquina virtual con sistema operativo ***Ubuntu Server*** 20.04 LTS <https://ubuntu.com/download/server> con las siguientes características: 

> - Memoria RAM 2G
> - 2 CPU
> - Almacenamiento 16 G
> - Una Tarjeta de Red
> - Acceso a Internet

 En este servidor luego de actualizarlo (update) se deben instalar un servidor ***DNS*** (***Bind***) y un servidor ***Web*** (***Nginx***).

 En ambos casos se procederá a, la configuración de los mismos incluyendo las capas de seguridad criptográfica. (***DNSSEC*** y ***SSL***)

En el caso de la configuración ***SSL*** se utilizara un certificado ***auto firmado***, por lo que se debe implementar un CA (***myCA***) con ***openssl***

## Detalle de la configuración 

Para el DNS se debe utilizar el dominio que le corresponde de acuerdo al siguiente detalle (por alumno)


