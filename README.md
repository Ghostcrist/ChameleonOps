# Aso — Master Ansible para openSUSE Leap 16.0

> **Plantilla inicial de integración, NO es un despliegue funcional de servicios.**
> Los 5 roles traen tareas `debug` temporales que los compañeros deberán
> sustituir por automatizaciones reales.

## Requisitos

- Master: openSUSE Leap 16.0 con Ansible, Git, OpenSSH.
- VM02 web, VM03 archivos, VM04 PostgreSQL en la misma red accesible.
- En cada destino: Python 3, servidor SSH, usuario `ansible` con permisos sudo.
- Clave pública SSH de la master instalada en `~ansible/.ssh/authorized_keys`.
- IP reales introducidas en el archivo `hosts`.

## Primeros comandos (en la carpeta del proyecto)

```bash
ansible --version
ansible-config dump --only-changed
ansible-inventory --graph
ansible all -m ping
ansible-playbook 00_conectividad.yml
ansible-playbook 01_permisos_sudo.yml --ask-become-pass
ansible-playbook site.yml --syntax-check
```

Usa `--limit vm02`, `--limit vm03` o `--limit vm04` para probar
cada máquina independientemente cuando sus claves `sudo` sean distintas.

## Cuando los roles estén implementados

```bash
ansible-galaxy collection install -r collections/requirements.yml
ansible-playbook site.yml --syntax-check
ansible-playbook site.yml --list-hosts
ansible-playbook site.yml --check --diff --ask-become-pass
ansible-playbook site.yml --limit vm02 --ask-become-pass
ansible-playbook site.yml --limit vm03 --ask-become-pass
ansible-playbook site.yml --limit vm04 --ask-become-pass
# Solo después de validar por host:
ansible-playbook site.yml --ask-become-pass
```

Nota: check mode es orientativo; algunos módulos no predicen todos los cambios.

## Contraseñas distintas en cada destino

Ansible `--ask-become-pass` solicita una sola contraseña para la ejecución.
Si las 3 VM tienen contraseñas sudo diferentes, ejecutar un host por vez
con `--limit`, o usar un archivo cifrado específico de host:

```bash
mkdir -p host_vars/vm02
ansible-vault create host_vars/vm02/vault.yml
```

Dentro del editor cifrado, asignar `ansible_become_password`.
Repetir por cada host y ejecutar con `--ask-vault-pass`.
**Jamás publicar la contraseña de Vault ni secretos en texto plano.**

## Reglas de trabajo en GitHub

- No subir claves privadas SSH, contraseñas, archivos de datos de MySQL/PostgreSQL ni configuraciones sensibles sin revisión.
- Cada compañero desarrolla sus roles en una rama `feature/…`.
- Antes de `git push`, revisar `git status`, `git diff --cached`.
- Si el ZIP antiguo llegó a GitHub con secretos, ROTAR esos secretos y limpiar el historial.

## Responsables

- VM01 (master): `ansible.cfg`, `hosts`, `site.yml`, Git, Vault y pruebas.
- VM02 (web): roles `web` y `mariadb`.
- VM03 (archivos): roles `samba` y `ftp`.
- VM04 (BD): rol `postgres`.
