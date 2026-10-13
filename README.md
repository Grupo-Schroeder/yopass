<p align="center">
  <img src="https://raw.githubusercontent.com/jhaals/yopass/master/logo/yopass-horizontal.png" alt="Yopass" width="420">
</p>

# Yopass — Compartir secretos de forma segura

Instancia autoalojada de [Yopass](https://github.com/jhaals/yopass) para compartir contraseñas, tokens de API y otros datos sensibles mediante enlaces temporales o de un solo uso.

Yopass está pensado para evitar enviar credenciales directamente por correo, chats o sistemas de tickets. La instancia se ejecuta con Docker Compose, usa Memcached como almacenamiento temporal y se publica mediante Traefik.

## ¿Cómo funciona?

1. **Cifrado en el navegador:** el secreto se cifra en el navegador antes de enviarse al servidor.
2. **Almacenamiento del contenido cifrado:** Yopass guarda el mensaje cifrado y su vencimiento. El servidor no recibe la clave de descifrado junto con el secreto.
3. **Creación del enlace:** Yopass genera un enlace que contiene la clave en el fragmento de la URL (`#...`). El fragmento no se envía al servidor en las solicitudes HTTP.
4. **Recepción y descifrado:** el destinatario abre el enlace; su navegador obtiene el contenido cifrado y lo descifra localmente.
5. **Consumo o vencimiento:** según las opciones elegidas, el secreto puede eliminarse después de la primera lectura o dejar de estar disponible al vencer.

> Un enlace de un solo uso limita nuevas lecturas, pero no impide que el destinatario copie, fotografíe o conserve el secreto una vez que lo ve.

## Arquitectura de esta instalación

```text
Usuario
  │
  ▼
Traefik (HTTPS)
  │ red traefik-public
  ▼
Yopass :1337
  │ red yopass-internal
  ▼
Memcached :11211
```

- **Yopass:** aplicación web y API. Escucha en el puerto interno `1337`.
- **Memcached:** almacena temporalmente los mensajes cifrados. No se publica en el host ni se conecta a Traefik.
- **Traefik:** recibe las solicitudes HTTPS y las dirige al puerto `1337` del contenedor.
- **Redes Docker:** Yopass está conectado a `traefik-public` para recibir tráfico del proxy y a `yopass-internal` para comunicarse con Memcached.

En la configuración destinada a Traefik no se publica `1337` con `ports`: el proxy se conecta al puerto interno del contenedor. La red externa `traefik-public` debe existir previamente.

## Puesta en marcha

Requisitos:

- Docker Engine y Docker Compose.
- Traefik conectado a la red Docker externa `traefik-public`.
- Un DNS para el nombre elegido y HTTPS configurado en Traefik.

Iniciar los servicios:

```bash
docker compose up -d
```

Consultar el estado:

```bash
docker compose ps
docker compose logs -f yopass
docker compose logs -f memcached
```

Detener los servicios:

```bash
docker compose down
```

El comando anterior elimina los contenedores, pero no borra imágenes descargadas. Memcached mantiene los datos en memoria: al reiniciar o recrear el contenedor pueden perderse secretos que todavía no vencieron.

## Configuración de Traefik

El servicio Yopass debe tener etiquetas equivalentes a estas:

```yaml
labels:
  - "traefik.enable=true"
  - "traefik.docker.network=traefik-public"
  - "traefik.http.services.yopass.loadbalancer.server.port=1337"
  - "traefik.http.routers.yopass.rule=Host(`yopass.test.lebensalud.com`)"
  - "traefik.http.routers.yopass.service=yopass"
```

Reemplazá `yopass.test.lebensalud.com` si tu instancia usa otro dominio. Estas etiquetas presuponen que Traefik ya tiene configurados el entrypoint HTTPS y los certificados para el dominio.

## Seguridad y buenas prácticas

- **Usar HTTPS siempre.** En producción, no publiques Yopass por HTTP abierto. TLS protege la aplicación y el contenido cifrado frente a modificaciones durante el tránsito.
- **Compartir el contexto por otro canal.** Si compartís una contraseña, enviá el usuario o las instrucciones por un canal y el enlace por otro cuando sea posible.
- **Verificar el destinatario.** Cualquiera que obtenga el enlace puede intentar recuperar el secreto.
- **Elegir una expiración corta.** Usá el plazo más breve compatible con el caso de uso y la opción de un solo uso cuando corresponda.
- **No asumir que hay control de acceso corporativo.** La edición comunitaria no ofrece autenticación de usuarios integrada en el flujo estándar.
- **No compartir secretos en el propio enlace por otros medios.** Evitá pegar la contraseña en tickets, logs o mensajes junto al enlace.
- **Tener en cuenta Memcached.** Es almacenamiento temporal y volátil; no es una base de datos persistente ni una copia de seguridad.
- **Mantener la imagen actualizada.** Para un entorno productivo, fijá una versión concreta de la imagen en lugar de depender indefinidamente de `latest`.

## Limitaciones

- Una vez mostrado, el destinatario puede copiar el secreto o tomar una captura.
- La eliminación tras una lectura no revoca copias que el destinatario haya hecho.
- Los secretos pendientes pueden desaparecer si Memcached se reinicia o pierde su contenido.
- La autenticación corporativa, auditoría avanzada y otras funciones de administración pueden requerir la licencia comercial de Yopass.

## Documentación oficial

- [Repositorio oficial de Yopass](https://github.com/jhaals/yopass)
- [Documentación de Yopass](https://yopass.se/docs/)
- [Imagen oficial en Docker Hub](https://hub.docker.com/r/jhaals/yopass)

Yopass es software de código abierto. Consultá el repositorio oficial para conocer la licencia, las opciones de configuración y las instrucciones actualizadas.
