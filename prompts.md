# Prompts

## Prompt 1

**Modelo:** Opus 5  
**Effort:** medium  
**Herramienta:** Claude Code

~~~
Trabaja únicamente en modo de solo lectura. No modifiques archivos, no instales dependencias, no ejecutes migraciones, no hagas commits, no cambies de rama y no realices operaciones remotas.

Haz únicamente estas dos comprobaciones:

1. Resume brevemente los últimos 5 commits del repositorio, indicando SHA corto y propósito aparente de cada uno.
2. Busca en todo el proyecto la palabra "comentario" y dime en qué archivos relevantes aparece, agrupándolos por backend, frontend, tests y documentación cuando corresponda.

No propongas cambios ni soluciones. El objetivo es únicamente comprobar que puedes consultar Git y buscar por todo el repositorio.
~~~

**Qué salió:** El login estaba expirado; tras ejecutar `/login`, Claude comprobó Git y búsqueda global correctamente y detectó que el repositorio tenía solo 2 commits, no 5.

---

## Prompt 2

**Modelo:** Opus 5  
**Effort:** medium  
**Herramienta:** Claude Code

~~~
Audita únicamente la feature de comentarios de este repositorio. No audites el sistema completo.

Trabaja sobre la rama actual y puedes crear o modificar únicamente:

docs/auditoria/comentarios.md

No modifiques código de producción, tests, configuración, dependencias, base de datos ni otros archivos. No hagas commits, push ni operaciones remotas.

Tu trabajo es producir una auditoría priorizada y verificable de esta única rebanada.

Para cada hallazgo debes comprobar personalmente la evidencia en el repositorio antes de escribirlo. No incluyas un hallazgo solo porque parezca probable.

Escribe una fila por hallazgo con exactamente estas seis columnas:

1. Qué es
2. Dónde está, incluyendo archivo y línea concreta que lo demuestra
3. Qué se rompe y quién lo nota
4. Cuántos sitios hay que tocar, contados y no estimados
5. ¿Se deshace con un revert? Sí o no, y si no, explica por qué
6. ¿Bloquea o desbloquea a otro hallazgo? Nombra el hallazgo; si no, escribe "A ninguno"

Reglas adicionales:

- Una frase para describir cada hallazgo.
- No combines dos problemas distintos en una misma fila.
- Si no puedes verificar una casilla, déjala explícitamente en blanco o marcada como no verificada; no la completes por inferencia.
- Distingue evidencia observada de interpretación.
- Deduplica hallazgos que sean manifestaciones del mismo problema transversal.
- No uses la severidad sugerida por una herramienta como criterio suficiente de prioridad.
- Para priorizar, considera daño, radio de impacto, reversibilidad y dependencias entre hallazgos.
- Si un hallazgo depende de otro, respeta esa precedencia aunque el dependiente parezca más grave.
- No propongas ni implementes soluciones.
- No corrijas ningún problema.
- No amplíes el alcance fuera de comentarios aunque encuentres problemas interesantes en otras partes.
- Conserva cualquier incertidumbre: una tabla incompleta pero verificable es preferible a una completa basada en suposiciones.

Antes de terminar, revisa que cada fila tenga evidencia concreta en archivo y línea.

Deja exclusivamente el resultado de la auditoría en:

docs/auditoria/comentarios.md

Cuando termines, dime cuántos hallazgos candidatos encontraste, cuántos quedaron realmente verificados y cuáles quedaron sin verificar. No modifiques ningún otro archivo.
~~~

**Qué salió:** Generó 27 candidatos y una primera tabla de 24 hallazgos. La revisión humana encontró varias relaciones de dependencia incorrectas, un hallazgo compuesto, una consecuencia no demostrada y una rama incorrecta en la cabecera.

---

## Prompt 3

**Modelo:** Opus 5  
**Effort:** medium  
**Herramienta:** Claude Code

~~~
Revisa únicamente docs/auditoria/comentarios.md a partir de estas observaciones de revisión humana independiente.

No modifiques ningún otro archivo, código de producción, tests, configuración, dependencias ni base de datos. No hagas commit, push ni operaciones remotas. Puedes usar comandos de solo lectura para verificar evidencia antes de corregir el documento.

Aplica estas correcciones:

1. La rama real es `auditoria-fp`, no `s6/start`.

2. H03:
   - conserva el hallazgo de que DELETE y PATCH usan `autor_email` como mecanismo de autorización;
   - elimina la afirmación de que está "bloqueado por H01";
   - expresa que H01 facilita obtener el email, pero H03 existe independientemente si el correo se conoce por otra vía.

3. H04:
   - conserva el patrón `select * from comentarios order by id desc limit 1`;
   - deja explícito que el riesgo de concurrencia está verificado por inspección, pero no reproducido en ejecución.

4. H11:
   - no mezcles ausencia de clave foránea y ausencia de índice como un solo hallazgo;
   - conserva H11 únicamente como problema de integridad referencial por ausencia de claves foráneas;
   - no conviertas la ausencia de índice en un nuevo hallazgo salvo que puedas demostrar un efecto concreto dentro de esta auditoría; si solo es riesgo de rendimiento no medido, déjalo fuera.

5. H12:
   - conserva el hallazgo de borrado físico y respuestas potencialmente huérfanas;
   - elimina la afirmación de que está bloqueado por H11;
   - indica, si corresponde, que está relacionado con H11 pero puede corregirse también desde lógica de aplicación.

6. H16:
   - conserva explícitamente su condición de riesgo parcialmente verificado: la ausencia de límite está observada, pero su impacto real no está medido.

7. H17:
   - corrige la consecuencia: el PATCH actual solo permite editar comentarios con `estado === 'pendiente'`;
   - no afirmes que un comentario aprobado puede reescribirse;
   - conserva únicamente la falta de trazabilidad de las ediciones que realmente permite el sistema.

8. H18:
   - clasifícalo explícitamente como riesgo arquitectónico/de mantenibilidad, no como fallo funcional demostrado;
   - no le atribuyas roturas que no estén verificadas.

9. H19:
   - actualiza el estado de la suite con el baseline humano ya ejecutado antes de la auditoría:
     `Tests 1 passed (1)`;
   - conserva que solo existe un test funcional de camino feliz.

10. H23:
    - conserva explícitamente que el riesgo de fecha/zona horaria está inferido del código pero no fue reproducido en ejecución.

11. H24:
    - elimínalo de la tabla. El uso de índices como `key` está observado, pero el fallo visual concreto descrito no está demostrado para estos componentes y no debe presentarse como hallazgo confirmado.

12. Recalcula el orden y los recuentos finales después de estas correcciones.
    - No renumeres artificialmente para ocultar que H24 fue descartado si mantener los identificadores existentes ayuda a la trazabilidad.
    - Distingue hallazgos completamente verificados, parcialmente verificados y descartados.

13. Conserva la regla de una sola cuestión por fila y no añadas hallazgos nuevos salvo que sean imprescindibles para corregir una contradicción existente.

Antes de guardar, verifica con comandos de solo lectura cualquier afirmación que mantengas sobre Git, especialmente si dices que el secreto de H02 ya existía en el commit inicial.

Modifica exclusivamente:

docs/auditoria/comentarios.md

Al terminar, resume exactamente qué filas cambiaste, cuál eliminaste y cuáles siguen parcialmente verificadas.
~~~

**Qué salió:** Corrigió la rama y los problemas señalados, retiró H24 y dejó 23 hallazgos: 19 completamente verificados y 4 parcialmente verificados.

---

## Prompt 4

**Modelo:** Opus 5  
**Effort:** medium  
**Herramienta:** Claude Code

~~~
Modifica únicamente docs/auditoria/comentarios.md.

No cambies ninguna fila de la tabla, ningún hallazgo, ningún recuento existente, ningún orden de prioridad ni ninguna otra sección del archivo.

Añade al final una nueva sección titulada:

## Parte B — Las tres líneas

Debajo, escribe exactamente estas tres líneas, sin ampliarlas ni explicarlas:

1. El agente devolvió 27 hallazgos candidatos; comprobé completamente 19 abriendo los archivos y revisando la evidencia.

2. No supe decidir la prioridad de H04 frente a otros hallazgos de seguridad: el daño potencial es alto, pero la condición de carrera no fue reproducida ni medí su frecuencia.

3. No pude verificar el desfase de fecha descrito en H23: el patrón de UTC sin zona está en el código, pero no observé el problema en ejecución.

No añadas conclusiones, comentarios ni texto adicional.

No modifiques ningún otro archivo. No hagas commit, push ni operaciones remotas.

Al terminar, muestra únicamente:
- qué archivo modificaste;
- las tres líneas añadidas;
- `git status --short`.
~~~

**Qué salió:** Añadió únicamente la sección Parte B al final de `docs/auditoria/comentarios.md`.
