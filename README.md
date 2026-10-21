# CaC para AAP 2.6 con credenciales cifradas

Configuración como código para Ansible Automation Platform 2.6. Los valores
sensibles van cifrados con Ansible Vault, con una clave distinta por torre, y
se aplican con `infra.aap_configuration`.

## Estructura

```
common/           organizaciones, equipos y definiciones de bootstrap (clave "master")
torre-linux/      una carpeta por torre, cada una con su clave
torre-windows/
torre-cloud/
playbooks/        bootstrap.yml, site_common.yml, site_torre.yml
ee-cac/           Containerfile del Execution Environment
platform/         manifiestos del operador de AAP (ejemplo)
```

En cada torre, `secrets.yml` va cifrado y el resto en claro:

| Fichero | Cifrado |
|---|---|
| `secrets.yml` | **sí**, con el vault-id de la torre |
| `credentials.yml` | no — referencia variables de `secrets.yml` |
| `inventories.yml`, `projects.yml`, `job_templates.yml` | no |

## Cómo funciona

Un push a la rama principal dispara el webhook y AAP ejecuta el workflow
`CaC - Despliegue Completo`:

```
push → webhook → sync del proyecto (git fetch) → workflow

   ┌──────────────────────────────┐
   │ CaC - Common (Orgs y Equipos)│  playbooks/site_common.yml
   └──────────────┬───────────────┘  credencial: Vault - Master
                  │ on success
     ┌────────────┼────────────┐
     ▼            ▼            ▼
  Torre        Torre        Torre       playbooks/site_torre.yml
  Linux        Windows      Cloud       credencial: la Vault de su torre
```

El nodo común va primero porque las torres necesitan que sus organizaciones
existan antes de colgar credenciales de ellas. Las tres torres corren en
paralelo.

Dentro de un nodo de torre, AAP convierte la credencial Vault adjunta en un
flag de línea de comandos, usando su *vault-id* como etiqueta:

```
ansible-playbook --vault-id linux@prompt -e @extravars playbooks/site_torre.yml
```

La extra var `torre: torre-linux` del job template es la que decide qué carpeta
se carga, porque todos los `vars_files` del playbook son `../{{ torre }}/...`.
Ansible descifra el `secrets.yml` de esa carpeta, resuelve las referencias
`{{ vault_... }}` de `credentials.yml` y el rol
`infra.aap_configuration.dispatch` crea o actualiza los objetos en AAP por su
nombre.

Cada job recibe **una sola clave** y carga **solo los ficheros que esa clave
abre**. Por eso el job de la torre Linux no puede leer los secretos de Windows:
no lleva su credencial adjunta.

## Preparar los secretos

```bash
cp common/secrets.yml.example      common/secrets.yml
cp torre-linux/secrets.yml.example torre-linux/secrets.yml
cp torre-windows/secrets.yml.example torre-windows/secrets.yml
cp torre-cloud/secrets.yml.example torre-cloud/secrets.yml

# rellenar valores y cifrar cada uno con su etiqueta
ansible-vault encrypt --vault-id master@prompt  common/secrets.yml
ansible-vault encrypt --vault-id linux@prompt   torre-linux/secrets.yml
ansible-vault encrypt --vault-id windows@prompt torre-windows/secrets.yml
ansible-vault encrypt --vault-id cloud@prompt   torre-cloud/secrets.yml
```

## Puesta en marcha

1. Ajustar en `common/bootstrap.yml`: `bootstrap_cac_repo` y `bootstrap_ee_image`.
2. Construir `ee-cac/Containerfile` y publicarlo en un registro accesible desde
   AAP. Las colecciones van horneadas en la imagen: un
   `collections/requirements.yml` del proyecto no se instala en el EE
   containerizado de AAP 2.6.
3. Crear a mano en AAP: credencial Vault `Vault - Master` (vault-id `master`),
   credencial SCM de solo lectura, y el proyecto con `scm_update_on_launch`.
4. Crear y lanzar el JT `CaC - Bootstrap` con `playbooks/bootstrap.yml`, el EE y
   `Vault - Master`. Crea organizaciones, equipos, las credenciales Vault de
   cada torre, los job templates y el workflow.
5. Registrar el webhook del repositorio contra el workflow
   `CaC - Despliegue Completo`, evento `push`.

## Alta o rotación de una credencial

El equipo edita su fichero cifrado con su propia clave, sin descifrarlo en disco:

```bash
ansible-vault edit --vault-id linux@prompt torre-linux/secrets.yml
```

añade la variable, la referencia desde `credentials.yml` y abre un pull request.
Quien revisa no necesita ninguna clave de Vault.

## Notas

- El vault-id es una etiqueta, no un control de acceso: lo que protege el
  fichero es su contraseña.
- `infra.aap_configuration` es idempotente por nombre y **nunca borra**. Para
  eliminar un objeto hay que marcarlo `state: absent`.
- `common/secrets.yml` contiene también las claves de las torres y la suya
  propia, para que el bootstrap pueda recrear las credenciales Vault en AAP.
