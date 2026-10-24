# Acuerdos de colaboración

1. Para cambios de un servicio, trabajar en su carpeta `roles/<servicio>`.
2. Mantener tareas en `tasks/main.yml`; variables ajustables en `defaults/main.yml`.
3. Guardar archivos dinámicos en `templates/*.j2`.
4. Reinicios condicionales en `handlers/main.yml`.
5. No escribir claves, passwords, secretos ni claves privadas en Git.
6. Comprobar sintaxis e idempotencia antes de pedir integrar la rama.
7. La master administra inventario, grupos, `site.yml` y resuelve conflictos.
