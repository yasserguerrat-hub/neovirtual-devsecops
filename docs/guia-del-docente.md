# Guía del docente

Todo lo que necesitas para dictar este laboratorio sin haber participado en su construcción. Léela completa una vez, haz el ensayo de la sección 8 y recién entonces prepara la clase.

## 1. Qué enseña y por qué

**Caso.** NeoVirtual quiere un pipeline CI/CD con seguridad integrada. Una auditoría interna encontró cuatro problemas:

1. Sin pruebas de seguridad automatizadas en el pipeline.
2. Mala gestión de dependencias y actualizaciones.
3. Poca documentación y trazabilidad para auditorías.
4. Monitorización insuficiente del entorno en producción.

**Idea central.** El mismo gestor de tareas Flask se despliega por dos pipelines de Jenkins. En uno no hay controles. En el otro cada etapa verifica algo y puede detener el despliegue. Los estudiantes atacan ambas versiones, rompen el pipeline a propósito y revisan la evidencia que queda para el auditor.

**Al terminar, el estudiante debería poder:**
- Explicar por qué una inyección SQL, un borrado por GET, un IDOR y una cuenta sin límite de intentos son fallas distintas, y qué control corrige cada una.
- Decir qué herramienta detecta cada tipo de problema (secretos, código, dependencias, imagen, aplicación en ejecución) y por qué ninguna sustituye a las otras.
- Responder «¿qué código corre en producción y quién lo verificó?» con evidencia y no de memoria.

**Duración sugerida:** 2 horas (agenda en la sección 6).

## 2. Mapa del repositorio

```
neovirtual-devsecops/
├── apps/
│   ├── taskmanager-sin-devsecops/   puerto 5001, vulnerable a propósito
│   └── taskmanager-con-devsecops/   puerto 5002, corregida + pipeline seguro
├── docs/
│   ├── clase-interactiva.html       material para proyectar (abrir en el navegador)
│   ├── demo-ataques.md              guion de ataques contra las apps reales
│   ├── auditoria-trazabilidad.md    hallazgo → control → evidencia
│   └── guia-del-docente.md          este documento
├── infra/                           Jenkins listo para el laboratorio
├── docker-compose.yml               levanta ambas apps sin Jenkins
└── README.md
```

El proyecto se entrega sin historial de Git. Inicialízalo (`git init -b main`, `git add -A`, `git commit`) y súbelo a tu propio remoto antes de crear los jobs.

## 3. Requisitos

| Qué | Para qué |
|---|---|
| Docker (Desktop en Windows/Mac) | Todo corre en contenedores; no hace falta Python en tu equipo |
| Jenkins en un contenedor llamado **`jenkins`** con el socket de Docker montado | Ejecutar los dos pipelines |
| Un repositorio Git remoto propio (GitHub, GitLab…) | Jenkins corre en un contenedor y clona desde una URL; no ve tu carpeta local |
| Navegador | Abrir `docs/clase-interactiva.html` |

El nombre `jenkins` del contenedor es obligatorio: el pipeline seguro usa `--volumes-from jenkins` para que las herramientas vean el código. Si tu contenedor se llama distinto, cambia `JENKINS_CONTAINER` en `apps/taskmanager-con-devsecops/Jenkinsfile`.

Jenkins incluido en el repositorio: `infra/docker-compose.jenkins.yml` construye `infra/jenkins/Dockerfile` (imagen `jenkins/jenkins:lts-jdk17` más el paquete `docker-cli`), con usuario root, socket `/var/run/docker.sock` montado y puertos 8080 y 50000.

```bash
docker compose -f infra/docker-compose.jenkins.yml up -d --build
```

Si usas otro Jenkins, compruébalo con `docker exec jenkins docker --version`. En Debian 13 el paquete `docker.io` instala el motor pero **no** el comando `docker`; hace falta `docker-cli`.

## 4. Las dos aplicaciones

Ambas son un gestor de tareas: login, panel con tareas por usuario, crear y borrar tareas, y una ruta `/admin`.

### 4.1 `taskmanager-sin-devsecops` (puerto 5001)

El código original de la evaluación, sin modificar (`app.py`, `create_db.py`). Credenciales: `admin` / `password` y `user` / `password`.

| Falla | Dónde | Qué ocurre |
|---|---|---|
| **Inyección SQL** | `/login`, líneas 33 a 36 de `app.py` | Si la contraseña contiene `' OR '`, la consulta se arma concatenando texto. Con `' OR '1'='1` entras como admin sin contraseña. |
| **CSRF** | `/delete_task/<id>` por GET | Una acción que modifica datos responde a GET y no pide token. Un `<img src=…/delete_task/1>` en cualquier página borra la tarea de la víctima. |
| **IDOR** | mismo endpoint | Borra por `id` sin comprobar de quién es la tarea. |
| **Autenticación débil** | `hash_password` | SHA-256 sin sal, sin MFA, intentos ilimitados. |
| **Debug activo** | `Dockerfile` (`FLASK_DEBUG=1`) | Expone la consola de Werkzeug en `/console`. |

Alrededor del código hay fallas de proceso que también forman parte de la demostración: dependencias antiguas con CVE (`Flask 2.0.1`, `Werkzeug 2.0.1`, `Jinja2 3.0.1`), imagen `python:3.9` completa, contenedor como root y un Jenkinsfile cuya etapa *Test* solo imprime un mensaje.

> **Matiz importante sobre XSS.** El enunciado original menciona XSS, pero ninguna de las dos apps es explotable: Jinja2 escapa la salida por defecto. Está bien decirlo en clase. Es un buen ejemplo de que no todo hallazgo es explotable y de que las pruebas sirven para confirmarlo. La app segura añade una CSP como segunda barrera.

### 4.2 `taskmanager-con-devsecops` (puerto 5002)

Es el código seguro de la evaluación con unas pocas adiciones.

| Control | Cómo está implementado |
|---|---|
| Inyección SQL | Consultas parametrizadas siempre; el usuario y el código MFA se validan con expresiones regulares (lista blanca) antes de tocar la base |
| CSRF | Token por sesión, comparado con `secrets.compare_digest`; borrado solo por POST; cookie `SameSite=Lax` |
| IDOR | `DELETE … WHERE id = ? AND user_id = ?` |
| Autenticación | Hash con sal (werkzeug), MFA TOTP obligatorio, bloqueo tras 5 fallos por 300 s, mensaje de error genérico, `session.clear()` al iniciar sesión |
| Control de acceso | `/admin` exige `role == admin` en la sesión firmada y responde 403 si no |
| Cabeceras | CSP, `X-Frame-Options`, `nosniff`, `Referrer-Policy`, `Permissions-Policy`, COOP, CORP, `Cache-Control: no-store` |
| Contenedor | Imagen multi-etapa, usuario sin privilegios (uid 10001), sistema de archivos de solo lectura, sin capabilities, `HEALTHCHECK` |

**Qué cambié respecto del código de la evaluación** (explícalo si alguien lo compara):
- `create_db.py` ya no trae contraseñas por defecto. Las toma de variables de entorno o genera una aleatoria.
- `app.py` registra eventos de seguridad en JSON, expone `/health` y añade cabeceras. Esas cabeceras se añadieron porque ZAP las marcó.
- Las dependencias se fijaron en versiones sin CVE (`Flask 3.1.3`, `Werkzeug 3.1.9`). La primera versión que fijé fue rechazada por `pip-audit`.
- `app.run` escucha en `127.0.0.1` por defecto; en el contenedor corre gunicorn.

**MFA.** Al crear la base se imprime un secreto TOTP por usuario (`docker logs taskmanager-secure | grep MFA`). Se carga en Google Authenticator, Authy o similar. Un código vale 30 segundos y se acepta una ventana de ±1.

## 5. El pipeline seguro, etapa por etapa

Archivo: `apps/taskmanager-con-devsecops/Jenkinsfile`. Las herramientas corren en contenedores efímeros de versión fija, por lo que Jenkins no necesita Python ni nada instalado.

| # | Etapa | Herramienta | Qué detecta | ¿Bloquea? | Evidencia |
|---|---|---|---|---|---|
| 1 | Trazabilidad | git | Registra build, commit, rama, imagen y fecha | n/a | `build-info.json` |
| 2 | Secretos | Gitleaks | Claves y contraseñas en el código | Sí | `gitleaks.json` |
| 3 | SAST | Bandit | Patrones inseguros en el código propio (SQL concatenado, `debug=True`) | Sí, severidad media o mayor | `bandit.json` |
| 4 | SCA | pip-audit | Dependencias con CVE conocidos | Sí | `pip-audit.json` |
| 5 | Pruebas | pytest + cobertura | 29 pruebas de seguridad, mínimo 80 % (hoy ≈97 %) | Sí | `junit.xml`, `coverage.xml` |
| 6 | Build imagen | docker | Construye y etiqueta con el commit | Sí | etiquetas OCI |
| 7 | Imagen + SBOM | Trivy | Vulnerabilidades HIGH/CRITICAL con parche; genera inventario CycloneDX | Sí | `trivy.json`, `sbom.cdx.json` |
| 8 | Deploy staging | docker | Despliega con secretos de Jenkins, solo lectura y sin capabilities | Sí | contenedor `taskmanager-secure` |
| 9 | Smoke test | healthcheck | Espera estado *healthy* | Sí | estado del contenedor |
| 10 | DAST | OWASP ZAP baseline | Prueba la app en ejecución, de forma pasiva y sin autenticar | **No**, marca UNSTABLE | `zap-report.html/json` |

**Por qué cada herramienta cubre algo distinto.** Si un estudiante pregunta «¿no basta con una?»: Gitleaks no ve CVE, Bandit no ve dependencias, pip-audit no ve el código propio, Trivy ve el sistema operativo de la imagen y ZAP ve cómo se comporta la app desde fuera. Es defensa en capas.

**Por qué ZAP no bloquea.** Es una decisión de política y no un límite técnico: el baseline es pasivo y sin autenticar, y sus avisos actuales son informativos. Cambiarlo es cuestión de quitar `catchError` en la etapa. Esa discusión suele salir sola.

**Pipeline sin DevSecOps** (`apps/taskmanager-sin-devsecops/Jenkinsfile`): Build, Test (vacío) y Deploy. Termina en verde en un minuto.

**Secretos.** El pipeline seguro lee tres credenciales de Jenkins (`taskmanager-secret-key`, `taskmanager-admin-pw`, `taskmanager-user-pw`) y las pasa al contenedor como variables de entorno sin escribirlas en la línea de comandos. Ningún secreto vive en el repositorio.

## 6. Agenda sugerida (120 min)

| Min | Bloque | Material |
|---|---|---|
| 0–10 | Caso y hallazgos de la auditoría | Página interactiva, secciones *Inicio* y *Escenario* |
| 10–25 | Levantar las dos apps (los estudiantes lo hacen) | Sección *Entorno*; `docker compose up -d --build` |
| 25–50 | Ataques contra la versión vulnerable, luego contra la segura | Simulador de la página para explicar; después `docs/demo-ataques.md` contra las apps reales |
| 50–55 | Pausa | |
| 55–75 | Crear los dos jobs y ejecutarlos | Sección *Jenkins*. Mientras corre el seguro, explica cada etapa con la tabla de la sección 5 |
| 75–95 | Romper el pipeline a propósito | Simulador *Pipeline* y luego, en vivo, el cambio de Flask a 2.0.1 |
| 95–110 | Evidencia para el auditor | Artefactos archivados del build; sección *Evidencia* |
| 110–120 | Cierre | Cuestionario de la página |

Consejo: el primer build seguro tarda varios minutos porque descarga imágenes (ZAP pesa más de 1 GB). Lánzalo antes de la clase o apenas empiece el bloque de ataques.

## 7. Guion de la demostración

Resultados esperados contra las apps reales (los comprobé yo):

| Ataque | Sin DevSecOps (:5001) | Con DevSecOps (:5002) |
|---|---|---|
| Login con usuario `admin` y contraseña `' OR '1'='1` | Entra al panel sin conocer la contraseña | `401` (consulta parametrizada); con un usuario como `admin' --` responde `400` |
| Abrir `/delete_task/1` con sesión | Borra la tarea (`302`) | `405`; un POST sin token da `400` |
| Abrir `/console` | `200`, consola de Werkzeug | `404` |
| Cinco o más contraseñas incorrectas | Sin límite | A la quinta se bloquea la cuenta y las siguientes dan `429` |

Los eventos de seguridad de la app segura se ven con:

```bash
docker logs taskmanager-secure
```
Busca `login_failure`, `account_locked`, `csrf_rejected` y `login_invalid_input`.

**Romper el pipeline en vivo.** En `apps/taskmanager-con-devsecops/requirements.txt` cambia `Flask==3.1.3` por `Flask==2.0.1`, haz commit y push, y ejecuta el job. Debe fallar en **SCA** y no llegar a Deploy. Revierte después. Otros cambios que se detienen: concatenar SQL en `app.py` (SAST), poner `debug=True` (SAST), commitear una clave (Secretos), quitar la verificación CSRF (Pruebas).

## 8. Ensayo previo (obligatorio, 30 minutos)

Las piezas del pipeline las probé una por una en Docker, pero **no ejecuté el Jenkinsfile completo dentro de Jenkins**. Haz una pasada completa antes de la clase:

1. Levanta Jenkins y crea las tres credenciales y los dos jobs (README, sección *Pipelines en Jenkins*).
2. Ejecuta el job sin DevSecOps: debe terminar verde y responder en http://localhost:5001.
3. Ejecuta el job seguro: debe recorrer las diez etapas, con DAST en amarillo, y responder en http://localhost:5002.
4. Revisa que el build tenga artefactos en `apps/taskmanager-con-devsecops/reports/`.
5. Haz el cambio de Flask a 2.0.1 y confirma que falla en SCA. Revierte.

## 9. Problemas frecuentes

| Síntoma | Causa y solución |
|---|---|
| `port is already allocated` en 5001 o 5002 | Tienes `docker compose up` activo y el pipeline intenta usar los mismos puertos. Ejecuta `docker compose down` en la raíz del repo antes de los jobs. |
| La etapa Secretos o SAST falla con «no such container: jenkins» o no encuentra archivos | El contenedor de Jenkins no se llama `jenkins`. Renómbralo o cambia `JENKINS_CONTAINER`. |
| `docker: command not found` dentro de Jenkins | La imagen no tiene el CLI de Docker o no monta `/var/run/docker.sock`. En Debian 13 el paquete `docker.io` no trae el comando `docker`; hace falta `docker-cli`. Usa `infra/docker-compose.jenkins.yml` y comprueba con `docker exec jenkins docker --version`. |
| Falla el deploy por credenciales | Los IDs deben ser exactamente `taskmanager-secret-key`, `taskmanager-admin-pw` y `taskmanager-user-pw`, tipo *Secret text*. |
| Cambié las contraseñas y la app segura sigue pidiendo las antiguas | La base vive en el volumen `taskmanager-data` y los usuarios existentes no se actualizan. Reinicia con `docker volume rm taskmanager-data` y vuelve a desplegar. |
| La etapa Trivy tarda mucho la primera vez | Descarga su base de datos. Se guarda en el volumen `trivy-cache`; las siguientes ejecuciones son rápidas. |
| ZAP falla con «directory /zap/wrk is not mounted» | Ya está resuelto en el Jenkinsfile con un volumen `zap-wrk`. Si modificas esa etapa, conserva el volumen y el `chown 1000:1000`. |
| El código MFA siempre es inválido | El reloj del equipo está desfasado, o cargaste el secreto del otro usuario. Los secretos de `admin` y `user` son distintos. |
| `pip-audit` falla en un build que antes pasaba | Aparecieron CVE nuevos. Es el comportamiento esperado de la compuerta: actualiza la versión afectada. |
| Un build seguro falla en Trivy sin cambios tuyos | Se publicó una vulnerabilidad nueva en la imagen base. Reconstruye con una base actualizada o ajusta la política con criterio. |

## 10. Preguntas que probablemente harán

**¿Por qué la app «sin DevSecOps» ya se parece a un caso real?** Porque las fallas de proceso (dependencias viejas, root, debug, test vacío) son tan comunes como las del código. Muestra que el problema no es solo escribir mal una consulta.

**¿Esto es seguro para producción?** No tal cual. Es un laboratorio: usa SQLite y HTTP. En producción irían una base gestionada, HTTPS con `COOKIE_SECURE=true`, un gestor de secretos y un SIEM real que consuma los logs JSON.

**¿Por qué SQLi si el código ya usa `?` en la rama normal?** Porque la vulnerabilidad está en la rama que concatena. Basta una ruta insegura para que toda la defensa falle.

**¿Para qué sirve el SBOM?** Es el inventario exacto de componentes de la imagen. Si mañana sale un CVE en una librería, sabes en qué versiones desplegadas está.

**¿Dónde queda la «monitorización»?** En tres capas: eventos JSON de seguridad en la app, `/health` con `HEALTHCHECK` de Docker y un smoke test al desplegar. Alertar sobre ellos requiere un colector externo, fuera del alcance de este laboratorio.

## 11. Qué no cubre este laboratorio

- Autenticación en ZAP, o pruebas dinámicas profundas.
- Un SIEM o sistema de alertas real.
- Gestión de secretos fuera de Jenkins (Vault, secret managers en la nube).
- Despliegue en un entorno distinto de un Docker local.

Son buenas extensiones para una segunda sesión.

## 12. Aviso

`apps/taskmanager-sin-devsecops` es vulnerable a propósito. Ejecútala solo en una máquina o red aislada y no la expongas a internet.
