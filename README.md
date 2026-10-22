# PHP2SHELL — Lab y script de POC

Laboratorio local para practicar con la vulnerabilidad de `unserialize()` en PHP 8.5.11
y su script de prueba de concepto.

> **Aviso:** este material es solo para fines educativos e investigación de seguridad autorizada.
> Úsalo únicamente en sistemas que sean tuyos o que tengas permiso explícito para probar.
> El lab expone endpoints de ejecución de código sin autenticación: no lo dejes en redes que no controlas
> y apágalo con `./run.sh down` al terminar.

## Estructura

```
LAB/
├── Dockerfile            PHP 8.5.11 + Apache, dos puntos de entrada, TLS
├── docker-compose.yml    red bridge, puertos 8486 (HTTP) y 8483 (HTTPS)
├── run.sh                up | status | down | clean
├── remote_app.php        punto de entrada C: (clase Serializable)
├── remote_app_enum.php   punto de entrada E: (autoload de enum)
└── check.php             detector seguro de ambas ramas
POC/
└── POC-PHP2SHELL.py      script de POC (remoto)
```

## Requisitos

| Herramienta        | Para                        | Probado con                 |
|--------------------|-----------------------------|-----------------------------|
| Python >= 3.8      | `POC/POC-PHP2SHELL.py`      | 3.14.5 (macOS arm64)        |
| Docker + Compose   | lab en `LAB/`               | Docker 29.6.2 / Compose v5.1.4 |

Comprueba las versiones:

```bash
python3 --version        # >= 3.8
docker compose version
```

## 1. Habilitar el lab

```bash
cd LAB
chmod +x run.sh
./run.sh up              # construye la imagen y levanta el contenedor
./run.sh status          # estado, URLs y comprobación con check.php
```

Por defecto el contenedor escucha en todas las interfaces del host:

- Local: `http://127.0.0.1:8486` y `https://127.0.0.1:8483` (certificado autofirmado)
- Red local: `http://IP_DE_TU_EQUIPO:8486`

Puedes cambiar los puertos con variables de entorno:

```bash
HTTP_PORT=9000 HTTPS_PORT=9443 ./run.sh up
```

Para apagarlo y eliminarlo:

```bash
./run.sh down            # detiene y elimina el contenedor
./run.sh clean           # además elimina la imagen
```

### Endpoints del lab

| Ruta               | Rama | Descripción                              |
|--------------------|------|------------------------------------------|
| `/remote_app.php`  | C:   | Clase `Serializable` incluida en la app  |
| `/enum.php`        | E:   | Autoload de enum, sin clase propia       |
| `/check.php`       | —    | Detector seguro para confirmar el estado |

Antes de ejecutar el POC, comprueba que el lab responde:

```bash
curl http://127.0.0.1:8486/check.php
```

## 2. Ejecutar el script de POC

Desde la carpeta `POC/`, apuntando al lab local:

```bash
python3 POC-PHP2SHELL.py --url http://127.0.0.1:8486/remote_app.php
```

Para el endpoint E: usa `--path /enum.php`, y para HTTPS añade `--insecure` (certificado autofirmado):

```bash
python3 POC-PHP2SHELL.py --url https://127.0.0.1:8483/enum.php --insecure
```

Si quieres probar desde otra máquina de tu red de laboratorio, sustituye `127.0.0.1` por la IP del equipo que ejecuta el lab:

```bash
python3 POC-PHP2SHELL.py --url http://IP_DEL_LAB:8486/remote_app.php
```

Ver todas las opciones:

```bash
python3 POC-PHP2SHELL.py --help
```

## Resultados verificados (php:8.5.11)

| Cadena           | Peticiones | Resultado                             |
|------------------|-----------:|---------------------------------------|
| C: HTTP          | 2.169      | RCE como `www-data`                   |
| C: HTTPS         | 2.169      | RCE como `www-data`                   |
| E: HTTP          | 2.281      | RCE como `www-data`                   |
| E: HTTPS         | 2.175      | RCE como `www-data`                   |
| C: / E: local    | —          | root, `disable_functions` eludido     |

## Solución de problemas

- **`PHP X != 8.5.11`** al hacer `up`: la imagen base no es la de referencia; el lab sigue funcionando pero no es el objetivo verificado.
- **El puerto está ocupado**: usa `HTTP_PORT` / `HTTPS_PORT` como se indicó arriba.
- **Un ataque deja Apache sin responder**: el lab tolera crashes (prefork con varios workers), pero si no responde, ejecuta `./run.sh up` de nuevo para reiniciar el contenedor.

## Créditos

- PoC original y autor: Karim BOUDRA @ ULTRADEFENSE LAB.
- Investigación: Calif (MAD Bugs, PoC público de la rama C:), Nicolas Grekas (GH-23981, rama E:).
- Saludos: CYBERSEC TEAM.
