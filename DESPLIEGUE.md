# DESPLIEGUE — Evidencias y respuestas

Este documento recopila todas las evidencias y respuestas de la practica.

---

## Parte 1 — Evidencias minimas

### Fase 1: Instalacion y configuracion

1) Servicio Nginx activo
- Que demuestra: El servicio Nginx se encuentra activo y funcionando correctamente dentro del contenedor Docker.
- Comando: ``docker compose ps`` y ``docker exec -it nginx-web service nginx status``
- Evidencia:

![Servicio_Nginx_Activo](evidencias/Captura%20de%20pantalla%202026-01-13%20232312.png)

![Servicio_Nginx_Activo_2](evidencias/Captura%20de%20pantalla%202026-01-14%20000045.png)


2) Configuracion cargada
- Que demuestra: La configuracion personalizada de Nginx ha sido correctamente cargada y esta siendo utilizada por el servidor web.
- Comando: ``docker exec -it nginx-web ls -l /etc/nginx/conf.d/``
- Evidencia:

![Configuracion_cargada](evidencias/Captura%20de%20pantalla%202026-01-14%20000125.png)

3) Resolucion de nombres
- Que demuestra: El nombre de dominio configurado como ``rocio-app`` en lugar de una direccion IP. Para llegar a este punto, primero configuré el archivo ``hosts`` en mi sistema operativo local, agregando la siguiente linea: ``127.0.0.1    rocio-app``.
- Evidencia:

![Resolucion_de_nombres](evidencias/Captura%20de%20pantalla%202026-01-14%20003832.png)

4) Contenido Web
- Que demuestra: El contenido web personalizado de Cloud Academy ha sido correctamente desplegado y es accesible a traves del navegador web.
- Evidencia:

![Contenido_web](evidencias/Captura%20de%20pantalla%202026-01-14%20003832.png)

### Fase 2: Transferencia SFTP (Filezilla)

5) Conexion SFTP exitosa
- Que demuestra: La conexion SFTP al servidor ha sido exitosa, permitiendo la transferencia segura de archivos.
- Evidencia:

![Conexion_SFTP_exitosa](evidencias/Captura%20de%20pantalla%202026-01-14%20005512.png)


6) Permisos de escritura
- Que demuestra: Para probar que puedo subir archivos, creo un archivo de prueba en mi ordenador (`prueba.txt`) y lo arrastro desde el panel izquierdo al panel derecho de Filezilla. En la primera captura se muestra el mensaje de transferencia exitosa, y en la segunda captura se puede ver el archivo `prueba.txt` en el directorio del servidor.
- Evidencia:

![Permisos_de_escritura](evidencias/Captura%20de%20pantalla%202026-01-14%20005835.png)

![Permisos_de_escritura_2](evidencias/Captura%20de%20pantalla%202026-01-14%20005843.png)


### Fase 3: Infraestructura Docker

7) Contenedores activos
- Que demuestra: Los contenedores Docker necesarios para el despliegue de la aplicacion estan activos y funcionando correctamente en los puertos ``0.0.0.0:8080->80/tcp`` y ``0.0.0.0:2222->22/tcp``.
- Comando: ``docker compose ps``
- Evidencia:

![Contenedores_activos](evidencias/Captura%20de%20pantalla%202026-01-13%20232312.png)


8) Persistencia (Volumen compartido)
- Que demuestra: Demuestra que lo que transfiero por SFTP se ve en la web. Primero creé un archivo HTML de prueba y lo subí a la carpeta `upload` del servidor a traves de Filezilla. Luego, accedi a ese archivo desde el navegador web utilizando la URL: `http://rocio-app:8080/prueba.html`.
- Evidencia:

![Persistencia_navegador](evidencias/Captura%20de%20pantalla%202026-01-14%20010601.png)

![Persistencia_servidor](evidencias/Captura%20de%20pantalla%202026-01-14%20010613.png)

![Persistencia_transferencia](evidencias/Captura%20de%20pantalla%202026-01-14%20011724.png)

9) Despliegue multi-sitio
- Que demuestra: El despliegue de un sitio web adicional en la ruta /reloj ha sido exitoso, mostrando un reloj digital en tiempo real. 
- Problema que encontré: Al acceder a `http://localhost:8080/reloj`, Nginx genera una **redirección 301** a `http://localhost/reloj/` (sin incluir el puerto 8080). El navegador intenta conectar entonces al puerto 80 (por defecto) donde no hay servicio, mostrando el mensaje de error: "localhost ha rechazado la conexión". Para solucionarlo, creé el archivo `config/default.conf` con la directiva `absolute_redirect off;` que hace que Nginx genere redirecciones **relativas** en lugar de absolutas, preservando así el puerto del cliente:

```bash
server {
    listen 80;
    listen [::]:80;
    
    server_name localhost rocio-app;
    
    # Evita redirecciones absolutas
    absolute_redirect off;

    root /usr/share/nginx/html;
    index index.html index.htm;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

Añado en el archivo ``docker-compose.yml`` el montaje de este archivo de configuración personalizado:

```yaml
    volumes:
      - ./web:/usr/share/nginx/html
      - ./config/default.conf:/etc/nginx/conf.d/default.conf:ro
``` 
- Evidencia:

![Despliegue_reloj](evidencias/Captura%20de%20pantalla%202026-01-15%20092903.png)


### Fase 4: Seguridad HTTPS

10) Cifrado SSL
- Que demuestra: La generación de certificados SSL autofirmados utilizando OpenSSL para habilitar conexiones seguras HTTPS en el servidor Nginx. Se muestra en las evidencias el mensaje de advertencia del navegador al tratarse de un certificado autofirmado, y el candado en la barra de direcciones que indica una conexión segura.
- Comando: 
```bash
  cd certificados
  openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout nginx-selfsigned.key -out nginx-selfsigned.crt
```

Posteriormente, configuré los archivos de `default.conf` y `docker-compose.yml` para habilitar HTTPS en Nginx.

- Evidencia:

![Comando_generacion_certificado](evidencias/Captura%20de%20pantalla%202026-01-14%20122448.png)

![Aviso_peligro](evidencias/Captura%20de%20pantalla%202026-01-14%20124121.png)

![Navegador_candado](evidencias/Captura%20de%20pantalla%202026-01-14%20124147.png)

11) Redireccion forzada
- Que demuestra: La redireccion forzada de todas las solicitudes HTTP a HTTPS utilizando una redireccion 301 en Nginx.
- Problema que encontré: Inicialmente, cuando intentaba acceder a `http://localhost:8080`, obtenía un error `ERR_SSL_PROTOCOL_ERROR`. La causa era que la redirección que generaba Nginx era incompleta. Si usaba `return 301 https://$host$request_uri;`, la variable ``$host`` solo devolvía ``localhost``, sin incluir el puerto 8080. Eso provocaba que la redirección se hiciera hacia ``https://localhost/`` (sin puerto), por lo que el navegador intentaba conectarse automáticamente al puerto 443. Como mi servicio HTTPS realmente está expuesto en el 8443, la conexión fallaba y aparecía el error de conexión rechazada.

Para solucionarlo, modifiqué la línea de redirección en `config/default.conf` para incluir explícitamente el puerto 8443:

```nginx
return 301 https://$host:8443$request_uri;
```

- Evidencia:

![Redireccion_forzada](evidencias/Captura%20de%20pantalla%202026-01-14%20170748.png)

![Redireccion_forzada_2](evidencias/Captura%20de%20pantalla%202026-01-14%20170817.png)

---

## Parte 2 — Evaluacion RA2 (a–j)

### a) Parametros de administracion
- Respuesta:
- Evidencias:
  - evidencias/a-01-grep-nginxconf.png
  - evidencias/a-02-nginx-t.png
  - evidencias/a-03-reload.png

### b) Ampliacion de funcionalidad + modulo investigado
- Opcion elegida (B1 o B2):
- Respuesta:
- Evidencias (B1 o B2):
  - evidencias/b1-01-gzipconf.png
  - evidencias/b1-02-compose-volume-gzip.png
  - evidencias/b1-03-nginx-t.png
  - evidencias/b1-04-curl-gzip.png
  - evidencias/b2-01-defaultconf-headers.png
  - evidencias/b2-02-nginx-t.png
  - evidencias/b2-03-curl-https-headers.png

#### Modulo investigado: <NOMBRE>
- Para que sirve:
- Como se instala/carga:
- Fuente(s):

### c) Sitios virtuales / multi-sitio
- Respuesta:
- Evidencias:
  - evidencias/c-01-root.png
  - evidencias/c-02-reloj.png
  - evidencias/c-03-defaultconf-inside.png

### d) Autenticacion y control de acceso
- Respuesta:
- Evidencias:
  - evidencias/d-01-admin-html.png
  - evidencias/d-02-defaultconf-auth.png
  - evidencias/d-03-curl-401.png
  - evidencias/d-04-curl-200.png

### e) Certificados digitales
- Respuesta:
- Evidencias:
  - evidencias/e-01-ls-certs.png
  - evidencias/e-02-compose-certs.png
  - evidencias/e-03-defaultconf-ssl.png

### f) Comunicaciones seguras
- Respuesta:
- Evidencias:
  - evidencias/f-01-https.png
  - evidencias/f-02-301-network.png

### g) Documentacion
- Respuesta:
- Evidencias: enlaces a todas las capturas

### h) Ajustes para implantacion de apps
- Respuesta:
- Evidencias:
  - evidencias/h-01-root.png
  - evidencias/h-02-reloj.png

### i) Virtualizacion en despliegue
- Respuesta:
- Evidencias:
  - evidencias/i-01-compose-ps.png

### j) Logs: monitorizacion y analisis
- Respuesta:
- Evidencias:
  - evidencias/j-01-logs-follow.png
  - evidencias/j-02-metricas.png

---

## Checklist final

### Parte 1
- [✅] 1) Servicio Nginx activo
- [✅] 2) Configuracion cargada
- [✅] 3) Resolucion de nombres
- [✅] 4) Contenido Web (Cloud Academy)
- [✅] 5) Conexion SFTP exitosa
- [✅] 6) Permisos de escritura
- [✅] 7) Contenedores activos
- [✅] 8) Persistencia (Volumen compartido)
- [✅] 9) Despliegue multi-sitio (/reloj)
- [✅] 10) Cifrado SSL
- [✅] 11) Redireccion forzada (301)

### Parte 2 (RA2)
- [ ] a) Parametros de administracion
- [ ] b) Ampliacion de funcionalidad + modulo investigado
- [ ] c) Sitios virtuales / multi-sitio
- [ ] d) Autenticacion y control de acceso
- [ ] e) Certificados digitales
- [ ] f) Comunicaciones seguras
- [ ] g) Documentacion
- [ ] h) Ajustes para implantacion de apps
- [ ] i) Virtualizacion en despliegue
- [ ] j) Logs: monitorizacion y analisis
