# wildfly-ansible

Configuración de WildFly por releases. Cada tag `vX.Y.Z` de este repositorio es una release: un
pipeline de AWS la recoge y la aplica **una sola vez** en las instancias EC2 elegidas por tags, con
el documento `AWS-ApplyAnsiblePlaybooks` de Systems Manager. No hay SSH ni inventario: el playbook
corre en local dentro de cada instancia.

Este repositorio solo dice **qué** se aplica. **Cómo** se aplica (el pipeline, el CodeBuild que
llama a SSM, los permisos) vive en el laboratorio `aws-wildfly` del workspace de preparación del
DOP-C02 (`Udemy-DevopsProfessionalCourse/laboratory/aws-wildfly/`, extensión Ansible).

## Qué hace el playbook

[`playbooks/wildfly-system-property.yml`](playbooks/wildfly-system-property.yml) añade a
`standalone.xml` la system property `demo.release`, con el tag de la release como valor:

1. **Para WildFly.** Mientras corre, WildFly es dueño de `standalone.xml` y lo reescribe en cada
   operación de gestión: un cambio hecho con el servidor en marcha se perdería.
2. **Añade la propiedad a `standalone.xml`** con `community.general.xml`.
3. **Levanta WildFly**, espera a que `/health` responda 200 y lee la propiedad en el servidor en
   marcha con `jboss-cli.sh --connect`.

Antes copia `standalone.xml` a `standalone.xml.<tag>.bak`. Si algo falla después de parar, el
`rescue` restaura la copia, arranca WildFly con la configuración anterior y da la release por
fallida. Si la propiedad ya tiene el valor de la release, no hace nada y **no para WildFly**.

Dos trampas de `standalone.xml` que el playbook resuelve, comprobadas con WildFly 35.0.1.Final:

- **El namespace cambia entre versiones** (en la 35 es `urn:jboss:domain:community:20.0`): se lee
  del propio elemento `<server>`.
- **WildFly exige un orden en los hijos de `<server>`**, y `system-properties` va justo detrás de
  `extensions`. Si el módulo crea el nodo por su cuenta, lo pone al final y WildFly no arranca
  (`WFLYCTL0198: Unexpected element`). Por eso se crea con `insertafter` detrás de `extensions`.

## Estructura

```text
playbooks/
├── wildfly-system-property.yml   # hosts: localhost · connection: local · become: true
└── vars/
    └── wildfly.yml               # rutas, servicio, puerto, health y nombre de la propiedad
```

| Variable | Valor | De dónde sale |
|---|---|---|
| `release_tag` | `vX.Y.Z` | `ExtraVariables` del documento de SSM; el playbook rechaza cualquier otra forma |
| `wildfly_home` | `/opt/wildfly` | el stack base del laboratorio |
| `wildfly_service` · `wildfly_user` | `wildfly` | ídem |
| `wildfly_http_port` · `wildfly_health_path` | `8080` · `/health` | ídem |
| `wildfly_property_name` | `demo.release` | este repositorio |

## Publicar una release

```bash
git tag v1.0.0
git push origin v1.0.0
```

- **Un push a `main` no despliega nada.** Solo dispara un tag que case con `v*`.
- **Una release no se repite.** El pipeline guarda una marca por tag y para cualquier segunda
  ejecución del mismo tag (un webhook reenviado, un tag borrado y vuelto a crear). Si una release
  falla, se corrige y se publica otra (`v1.0.1`).
- **Protege los tags** con un *ruleset* de GitHub sobre `v*` que impida moverlos o borrarlos: si un
  tag puede apuntar mañana a otro commit, deja de identificar una release.

## Cómo lo ejecuta AWS

El agente de SSM descarga el repositorio **en el commit exacto del tag** (`getOptions:
commitID:<sha>`) y ejecuta:

```bash
ansible-playbook -i "localhost," -c local -e "release_tag=vX.Y.Z" -v \
  playbooks/wildfly-system-property.yml
```

Con `InstallDependencies=True`, el documento instala Ansible con `pip3 install ansible --upgrade`.
En Amazon Linux 2023 (Python 3.9) eso da `ansible 8.7.0` (`ansible-core 2.15.13`), comprobado el 4
de octubre de 2026. La versión **no queda fijada**: cada ejecución vuelve a mirar PyPI. `lxml`, que
necesita `community.general.xml`, la instala el propio playbook como paquete del sistema
(`python3-lxml`).

## Probarlo en local

Sin systemd, en un contenedor de Amazon Linux 2023 con WildFly en `/opt/wildfly`, se prueba solo la
edición del XML (los pasos que necesitan systemd llevan el tag `service`):

```bash
ansible-playbook -i "localhost," -c local -e "release_tag=v1.0.0" --skip-tags service \
  playbooks/wildfly-system-property.yml
```
