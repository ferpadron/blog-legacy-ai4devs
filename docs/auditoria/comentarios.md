# Auditoría de la feature de comentarios — blog «Corriente»

**Alcance:** exclusivamente la rebanada de comentarios: crear (`POST /posts/:slug/comentarios`),
editar (`PATCH /comentarios/:id`), borrar (`DELETE /comentarios/:id`) y listar/contar comentarios
(dentro de `GET /posts/:slug`, `GET /posts` y `GET /posts/:slug/relacionados`), más la tabla
`comentarios`, su semilla, su test y los componentes de front que los consumen.

Quedan **fuera** de esta auditoría (vistos pero no auditados): duplicación de nombre/correo del
autor y del nombre de categoría en `posts`, `posts.etiquetas` como cadena con comas, el N+1 de
`GET /etiquetas`, los filtros de `GET /posts`, el `estado = 'borrador'` de posts y la copia
repetida de la URL base de la API en el front.

**Rama:** `auditoria-fp` · **Fecha:** 2026-10-01 · **Modo:** solo lectura.

## Cómo leer la tabla

- **Evidencia observada** = leída en el archivo y línea citados, o medida contra
  `backend/tmp/db.sqlite3` en modo `ro`, o comprobada con comandos de git de solo lectura. Se marca `[obs]`.
- **Interpretación** = consecuencia deducida de esa lectura, no ejecutada en el sistema. Se marca `[int]`.
- Las casillas que no he podido comprobar se dejan como **no verificado**, no se rellenan por inferencia.
- Prioridad: daño × radio de impacto, corregida por reversibilidad y por precedencia de dependencias.
  Ninguna fila usa la severidad de una herramienta como criterio: no se ha ejecutado ninguna.

### Mediciones de apoyo (solo lectura)

```
sqlite3 'file:backend/tmp/db.sqlite3?mode=ro' "select estado, count(*) from comentarios group by estado"
  Aprobado|2   aprobado|4   ok|1   pendiente|2        (9 filas en total)

post_id = 1 ('diez-tendencias-de-diseno-que-pueden-cambiar-la-web-moderna'):
  recuento del listado  (estado <> 'pendiente')  -> 4
  comentarios del detalle (estado = 'aprobado')  -> 2  (ids 1 y 2; el 2 es respuesta del 1)
  hilos raíz que pinta el front                  -> 1

select sql from sqlite_master where tbl_name='comentarios'
  -> sin FOREIGN KEY y sin CHECK sobre `estado`

git show e0e0943:backend/start/routes.ts | grep dev-secret-cambiar   -> línea 25
git show origin/main:backend/start/routes.ts | grep dev-secret-cambiar -> línea 25
git merge-base --is-ancestor e0e0943 origin/main                     -> sí
git log --oneline --follow -- backend/start/routes.ts                -> solo e0e0943
```

Baseline de la suite aportado por la revisión humana, ejecutado antes de esta auditoría:
`Tests 1 passed (1)`. No lo he ejecutado yo; lo doy por evidencia de terceros, no propia.

## Tabla de hallazgos

| # | 1. Qué es | 2. Dónde está (archivo:línea que lo demuestra) | 3. Qué se rompe y quién lo nota | 4. Sitios a tocar (contados) | 5. ¿Revert lo deshace? | 6. ¿Bloquea o desbloquea a otro hallazgo? |
|---|---|---|---|---|---|---|
| H01 | Las respuestas públicas de comentarios devuelven la fila cruda, incluido `autor_email`. | `backend/start/routes.ts:152` y `:171` (detalle), `:329`–`:333` (creación), `:389`–`:392` (edición): `select *` sin filtrar y devuelto tal cual `[obs]` | Se publica el correo de cualquiera que comente a todo visitante anónimo; lo notan las personas que comentaron, y lo nota la regla 6 del README:65 `[obs]`. Nadie en el equipo se enteraría sin leer la respuesta `[int]` | 3 endpoints / 6 líneas: `routes.ts` 152, 171, 329, 333, 389, 392 | Sí: el cambio vive en un solo archivo, sin migración ni dato derivado | Desbloquea a H20 (la aserción negativa que lo cazaría no existe). Facilita la explotación de H03 al publicar los correos, pero no es condición necesaria de H03 |
| H02 | La clave que autoriza moderar está escrita en el repositorio como constante. | `backend/start/routes.ts:25` (`const SECRETO_SESION = 'dev-secret-cambiar'`), usada como credencial en `:351` `[obs]` | Cualquiera con acceso al repositorio —o que pruebe ese valor por defecto— borra cualquier comentario con una cabecera `x-clave-moderacion`; lo nota el autor del comentario al verlo desaparecer `[int]` | 1 declaración (`:25`) + 1 uso (`:351`) = 2 líneas. El `TODO(2024-11)` de `:22`–`:23` ya lo reconoce `[obs]` | No del todo: revertir el código restaura el comportamiento, pero el valor ya está publicado en el historial. Comprobado: está en `e0e0943` y en `origin/main`, y `e0e0943` es ancestro de `origin/main` `[obs]` | A ninguno |
| H03 | El borrado y la edición se autorizan con el `autor_email` que envía quien hace la petición. | `backend/start/routes.ts:350`–`:353` (DELETE compara contra `request.input('autor_email')`), `:378`–`:379` (PATCH, igual) `[obs]` | Quien conozca un correo borra o edita comentarios ajenos sin autenticarse; lo nota la persona suplantada y la moderación `[int]` | 2 endpoints / 4 líneas: `routes.ts` 350, 353, 378, 379 | Sí para el código. No para los datos: un comentario ya borrado por esta vía no vuelve (`:359` es `delete`, no borrado lógico) `[obs]` | A ninguno. El defecto es el mecanismo en sí y existe de forma independiente en cuanto el correo se conozca por cualquier vía —la firma de un comentario, una lista de distribución, un volcado anterior—. H01 solo abarata obtenerlo |
| H04 | Tras insertar, se devuelve el último comentario de **toda** la tabla, no el recién creado. | `backend/start/routes.ts:329`–`:331`: `select * from comentarios order by id desc limit 1`, sin filtrar por `post_id` ni usar el id insertado `[obs]` | El patrón está verificado por inspección del código `[obs]`; la consecuencia —que con dos inserciones concurrentes quien comenta reciba el comentario de otra persona, con su correo, como si fuera el suyo— es **interpretación no reproducida en ejecución** `[int]`. Lo notaría quien comenta | 1 sitio: `routes.ts:329`–`:331` | Sí: no deja rastro en la base de datos (el `insert` de `:324` es correcto) | A ninguno |
| H05 | La columna `comentarios.estado` es texto libre, sin `CHECK` ni enum, y ya conviven tres valores para «aprobado». | `backend/database/migrations/1769000000005_create_comentarios_table.ts:18` (`table.string('estado')`) `[obs]`; valores reales medidos: `aprobado` 4, `Aprobado` 2, `ok` 1, `pendiente` 2 `[obs]`; sembrados a propósito en `backend/database/seeders/blog_seeder.ts:510`, `:522`, `:575` `[obs]` | Nada se rompe por sí solo: es la causa de que el recuento (H06) y el listado (H07) discrepen. Lo nota el lector del blog a través de esos dos `[int]` | 1 columna (migración `:18`) + 3 filas de semilla (`:510`, `:522`, `:575`) + 3 filas vivas en `backend/tmp/db.sqlite3` | No: una vez normalizados los valores, revertir el código deja los datos ya reescritos. Revertir solo la migración implica recrear la tabla | **Bloquea a H06 y a H07**: mientras `estado` admita `'Aprobado'` y `'ok'`, cualquier comparación por igualdad o por desigualdad sigue siendo ambigua |
| H06 | El recuento de comentarios cuenta por exclusión (`<> 'pendiente'`), no por aprobación. | `backend/start/routes.ts:75`–`:76` (listado) y `:223` (relacionados) `[obs]` | El contador infla el número: para `post_id = 1` dice **4** cuando aprobados exactos hay 2 `[obs]`. Contradice la regla 5 del README:64 `[obs]`. Lo nota cualquier lector de la portada vía `frontend/src/componentes/ListaPosts.jsx:57` y `PostsRelacionados.jsx:32` `[obs]` | 2 consultas: `routes.ts:75`–`:76` y `:223`. Los 2 consumos de front (`ListaPosts.jsx:57`, `PostsRelacionados.jsx:32`) solo pintan el número | Sí: es SQL, no muta nada | Bloqueado por H05. Desbloquea a H21 (la divergencia de recuentos del front se alimenta de este número) |
| H07 | El detalle filtra los comentarios por igualdad exacta `estado = 'aprobado'`. | `backend/start/routes.ts:153` `[obs]`; el propio comentario de `:150`–`:151` lo admite `[obs]` | Comentarios aprobados guardados como `'Aprobado'` u `'ok'` no se muestran nunca: 3 de los 9 que hay `[obs]`. Lo notan sus autores (su comentario fue aprobado y no aparece) y el lector, que ve «4 comentarios» y un solo hilo `[obs]` | 1 consulta: `routes.ts:153` | Sí | Bloqueado por H05. Desbloquea a H21 |
| H08 | Borrar y editar aplican reglas de autorización distintas, escritas por separado. | `backend/start/routes.ts:349`–`:357` (DELETE: acepta clave de moderación, compara el correo en minúsculas y con `trim`) frente a `:378`–`:381` (PATCH: no acepta clave, compara el correo literal y además exige `estado === 'pendiente'`) `[obs]`; el comentario de `:375`–`:377` lo reconoce `[obs]` | Quien comentó con «Ana@Ejemplo.es» puede borrar su comentario pero no editarlo, y la moderación puede borrar pero no editar; lo nota quien comenta y lo nota la moderación `[int]` | 2 bloques / 11 líneas: `routes.ts:349`–`:357` y `:378`–`:381` | Sí | A ninguno. Comparte el mecanismo con H03, pero esta fila es solo la divergencia entre las dos copias de la regla |
| H09 | No existe ningún endpoint ni ruta que cambie el `estado` de un comentario. | `backend/start/routes.ts:326` inserta siempre `'pendiente'` `[obs]`; `grep -n "estado" backend/start/routes.ts` filtrado por `update`/`set` no devuelve ninguna línea `[obs]`; los endpoints documentados en `README.md:50`–`:52` son crear, borrar y editar `[obs]` | Un comentario nuevo no puede aprobarse por la API: nace `pendiente` y ahí se queda, invisible (H07). Lo nota quien comenta (su comentario nunca aparece) y la moderación, que no tiene herramienta `[int]`. Los aprobados que hoy se ven entraron por la semilla `[obs]` | 0 endpoints existentes que lo hagan; 1 archivo donde no está (`routes.ts`) | No aplica: no hay nada que revertir, es una ausencia | A ninguno |
| H10 | `padre_id` se inserta tal cual, sin comprobar que el comentario padre exista ni que pertenezca al mismo post. | `backend/start/routes.ts:308` (`request.input('padre_id', null)`) y `:326` (se pasa al `insert` sin validar) `[obs]`; sin restricción en la migración `:20` `[obs]` | Se puede colgar una respuesta de un comentario de otro post o de un id inexistente; el front filtra por `r.padre_id === c.id` (`frontend/src/componentes/Comentarios.jsx:64`) y por `!c.padre_id` (`:50`), así que esa respuesta no se pinta en ninguna parte y desaparece en silencio: `[obs]` del filtro, `[int]` de la consecuencia. Lo nota quien respondió | 1 endpoint / 2 líneas (`routes.ts:308`, `:326`) + 1 columna sin restricción (migración `:20`) | Sí para el código. No para los datos que se hubieran insertado mal. Hoy no hay ninguno: la consulta de huérfanos y la de padre en otro post devuelven 0 filas `[obs]` | A ninguno |
| H11 | La tabla `comentarios` no declara ninguna clave foránea: ni `post_id` → `posts.id` ni `padre_id` → `comentarios.id`. | `backend/database/migrations/1769000000005_create_comentarios_table.ts:14` (`table.integer('post_id')`) y `:20` (`table.integer('padre_id')`), sin `.references()` `[obs]`; el `CREATE TABLE` real de `sqlite_master` no contiene `foreign key` `[obs]`; el propio comentario de la migración `:6` lo admite `[obs]` | La integridad referencial queda enteramente en manos de la aplicación, que hoy no la comprueba (H10): un `post_id` o un `padre_id` inexistente se acepta en la tabla `[obs]` del esquema, `[int]` de la consecuencia. Lo notaría quien consulte los datos directamente. Con las 9 filas actuales no hay ninguna violación `[obs]` | 2 columnas en 1 migración (`:14`, `:20`) | No: en SQLite añadir una clave foránea exige recrear la tabla, así que revertir no devuelve el esquema anterior sin una segunda migración | A ninguno. H10 y H12 son los huecos de aplicación que esta ausencia deja sin red; ninguno de los dos depende de corregir el esquema |
| H12 | El borrado es físico y no toca las respuestas colgadas del comentario borrado. | `backend/start/routes.ts:359` (`delete from comentarios where id = ?`, sin tocar `padre_id` ni los hijos) `[obs]` | Borrar un comentario raíz deja sus respuestas con un `padre_id` que ya no existe; el front las excluye de los hilos (`Comentarios.jsx:50`, `:64`), así que se pierden de la vista sin borrarse: `[obs]` del filtro, `[int]` de la consecuencia. Lo nota quien respondió | 1 sitio: `routes.ts:359` | No: lo borrado no se recupera con un revert | A ninguno. Está relacionado con H11 —una clave foránea con regla de borrado definiría el comportamiento en la base de datos— pero es corregible también desde la lógica de aplicación, en esta misma línea, sin tocar el esquema |
| H13 | Un comentario rechazado por ser demasiado corto responde `200` con `{ ok: false }`. | `backend/start/routes.ts:319`–`:321` `[obs]`; el front lo trata leyendo `d.motivo` en `frontend/src/componentes/Comentarios.jsx:44` `[obs]` | Un rechazo viaja como éxito HTTP: cualquier cliente, caché o integración que mire el código de estado cree que el comentario se guardó. El front propio se salva porque comprueba `d.comentario` (`Comentarios.jsx:37`) `[obs]` | 1 línea de backend (`routes.ts:320`) + 1 rama de front que la consume (`Comentarios.jsx:44`) = 2 | Sí | A ninguno (es una cuestión distinta de H14: aquí el código de estado, allí la forma del cuerpo) |
| H14 | El mismo endpoint devuelve cuatro formas distintas de error. | `backend/start/routes.ts:311` (`{ error }`, 400), `:314` (`{ message }`, 422), `:317` (`{ errores: [] }`, 400), `:320` (`{ ok, motivo }`, 200) `[obs]`; el front encadena tres de ellas en `frontend/src/componentes/Comentarios.jsx:44` `[obs]` | Ningún cliente puede leer el error de forma uniforme; lo nota quien integra contra la API y quien comenta, que ve «No se ha podido enviar.» cuando la forma no encaja `[int]` | 4 respuestas en 1 endpoint (`routes.ts` 311, 314, 317, 320) + 1 línea de front que las absorbe (`Comentarios.jsx:44`) = 5 | Sí | A ninguno |
| H15 | La edición no aplica la longitud mínima que sí exige la creación. | `backend/start/routes.ts:384` (PATCH: solo `if (!cuerpo)`) frente a `:319` (POST: `length < 5`) `[obs]` | Un comentario pendiente se puede dejar en un solo carácter editándolo, y si después se aprueba entra así; lo notaría el lector `[int]` | 1 sitio: `routes.ts:384` | Sí para el código. No para los cuerpos ya reescritos: `:388` sobrescribe `cuerpo` sin guardar el anterior `[obs]` | A ninguno |
| H16 | No hay longitud máxima ni límite de frecuencia en la creación de comentarios. **Riesgo parcialmente verificado.** | `backend/start/routes.ts:305`–`:321`: las únicas comprobaciones son nombre presente, email presente, email con `@` y cuerpo de 5 o más caracteres `[obs]`; `cuerpo` es `text` sin límite (migración `:17`) `[obs]` | La **ausencia de límite está observada** `[obs]`; su **impacto real no está medido**: no he ejercitado el endpoint, así que el efecto de un cuerpo arbitrariamente grande o de muchas peticiones seguidas queda **no verificado**. Lo notaría quien opere la base de datos, y el lector si el spam llega a aprobarse. La semilla ya incluye un caso de spam (`blog_seeder.ts:553`, `estado: 'pendiente'`) `[obs]` | 1 bloque de validación (`routes.ts:305`–`:321`) + 1 columna sin límite (migración `:17`) | Sí para el código; no para las filas que se hubieran insertado | A ninguno |
| H17 | Editar un comentario no deja rastro: la tabla no tiene `updated_at` ni marca de edición. | Migración `1769000000005_create_comentarios_table.ts:11`–`:21`: las columnas son `id`, `post_id`, `autor_nombre`, `autor_email`, `cuerpo`, `estado`, `creado_en`, `padre_id`; no hay ninguna de actualización `[obs]`; `backend/start/routes.ts:388` actualiza solo `cuerpo` `[obs]` | El alcance real es el que permite el PATCH actual, que exige `estado === 'pendiente'` (`routes.ts:379`) `[obs]`: un comentario **pendiente** se puede reescribir cuantas veces se quiera y nadie puede saber que cambió ni cuándo, así que quien lo apruebe después lo aprueba sin saber si el texto es el original `[int]`. Lo nota la moderación. Un comentario ya aprobado **no** es editable por este endpoint `[obs]` | 1 migración (`:11`–`:21`) + 1 `update` (`routes.ts:388`) | No: añadir o quitar una columna exige otra migración, y las ediciones ya hechas no se reconstruyen | A ninguno |
| H18 | Los tres endpoints de comentarios viven enteros en el archivo de rutas, sin ninguna de las capas del proyecto. **Riesgo arquitectónico / de mantenibilidad, no fallo funcional demostrado.** | `backend/start/routes.ts:295`–`:393` (SQL, validación, autorización y forma de la respuesta en el handler) `[obs]`; `find backend/app -type f` no devuelve ningún controlador, servicio, validador, transformer ni modelo de comentarios: solo los de autenticación y `user.ts` `[obs]`; `backend/package.json` → `imports` no declara `#repositories/*` `[obs]` | No le atribuyo ninguna rotura: el comportamiento observado de los endpoints es el que describen las demás filas. Lo observable aquí es que cada regla está escrita una vez por endpoint, y que dos de esas copias ya no coinciden entre sí (H08, H15) `[obs]`. A quien lo nota es a quien mantiene el código `[int]` | 3 rutas / 99 líneas (`routes.ts:295`–`:393`); 0 archivos existentes en `backend/app/` para esta rebanada | Sí mientras no se mueva nada; a partir del primer archivo nuevo, el revert es un revert de varios archivos | No bloquea a ninguno. Es el archivo y el bloque donde están escritos H01, H03, H04, H06, H07, H09, H10, H12, H13, H14, H15 y H16 |
| H19 | La matriz de comportamiento de comentarios tiene un solo test, de camino feliz. | `backend/tests/functional/comentarios.spec.ts:15`–`:43`: un único `test`, que crea un comentario válido `[obs]`; su propia cabecera (`:5`–`:11`) enumera lo que no cubre `[obs]`. Baseline aportado por la revisión humana, ejecutado antes de esta auditoría: `Tests 1 passed (1)` `[obs, evidencia de terceros]` | Sin red: la suite está en verde con un solo test, así que nada caza una regresión en borrado, edición, listado de comentarios del detalle, email inválido, cuerpo corto ni post inexistente. Lo nota quien toque cualquiera de los hallazgos anteriores `[int]` | 1 archivo, 1 test, para 3 endpoints de escritura + 3 lecturas que exponen comentarios (`routes.ts:171`, `:75`, `:223`) | No aplica: es una ausencia de cobertura, no un cambio que revertir | A ninguno |
| H20 | El único test afirma un solo campo y ninguna ausencia. | `backend/tests/functional/comentarios.spec.ts:41`–`:42`: `assertStatus(201)` y una igualdad sobre `autor_nombre` `[obs]` | La fuga de H01 pasa el test en verde: no hay ninguna aserción negativa sobre `autor_email` ni sobre el resto del contrato. Lo nota quien confíe en que la suite protege el contrato `[int]` | 1 test, 2 aserciones (`comentarios.spec.ts:41`, `:42`) | No aplica: ausencia | Bloqueado por H01 en el orden de valor: la aserción que falta es precisamente la que demostraría H01 |
| H21 | El front mantiene dos recuentos de comentarios independientes, y uno de ellos se incrementa en local. | `frontend/src/componentes/DetallePost.jsx:30`, `:38`, `:71` (recuento de la cabecera) y `frontend/src/componentes/Comentarios.jsx:11`, `:23`, `:39`, `:54` (recuento del bloque, con `setTotal(total + 1)` al enviar) `[obs]`; los comentarios de `DetallePost.jsx:29` y `Comentarios.jsx:8`–`:10` lo admiten `[obs]` | Al enviar un comentario, el bloque dice uno más y la cabecera no se mueve: dos cifras distintas en la misma pantalla, y ninguna coincide con el `num_comentarios` del listado (4 frente a 2 medidos) `[obs]`. Lo nota cualquier lector que comente | 2 componentes / 7 líneas: `DetallePost.jsx` 30, 38, 71 y `Comentarios.jsx` 11, 23, 39, 54 | Sí: solo estado de React | Bloqueado por H06 y H07: mientras el recuento del servidor y la lista del detalle usen criterios distintos, cuadrar las cifras del front no es posible |
| H22 | Para mostrar los comentarios se vuelve a pedir el post completo, que la pantalla ya ha pedido. **Coste no medido.** | `frontend/src/componentes/Comentarios.jsx:19` (`fetch(API + '/posts/' + slug)`) y `frontend/src/componentes/DetallePost.jsx:34` (el mismo `fetch`), con `DetallePost.jsx:123` montando `<Comentarios slug={slug} />` `[obs]`; no existe ningún `GET` de comentarios (`README.md:45`–`:52` lista los endpoints) `[obs]` | La petición duplicada está observada en el código `[obs]`; que el lector lo note depende de la red y **no está medido** `[int]`. La consulta repetida incluye autor, categoría, etiquetas y anterior/siguiente | 2 peticiones en 2 componentes (`Comentarios.jsx:19`, `DetallePost.jsx:34`); 0 endpoints de lectura de comentarios | Sí | A ninguno |
| H23 | La fecha del comentario se construye a mano como cadena, sin zona horaria. **Inferido del código, no reproducido.** | `backend/start/routes.ts:323` (`new Date().toISOString().slice(0, 19).replace('T', ' ')`, es decir UTC sin marca de zona) `[obs]`; el front la recorta con `String(c.creado_en).substring(0, 10)` en `frontend/src/componentes/Comentarios.jsx:59` y `:68` `[obs]` | El patrón —se guarda UTC sin zona y se muestra esa cadena recortada— está observado `[obs]`. Que un comentario escrito de noche en España se muestre con el día anterior es **interpretación del código, no reproducida en ejecución**: no he ejercitado el endpoint para observar el desfase `[int]`. Lo notaría el lector | 1 escritura (`routes.ts:323`) + 2 lecturas de front (`Comentarios.jsx:59`, `:68`) = 3 | Sí para el código; no para las fechas ya escritas en la tabla | A ninguno |

## Orden de prioridad y por qué

1. **H01, H02, H03, H04** — daño directo a personas y radio máximo (cualquier anónimo). El orden
   dentro del grupo es por daño, no por dependencia: H03 va aquí por sí mismo, porque el mecanismo
   de autorización es un correo autodeclarado. H02 es el único hallazgo cuyo revert no basta, y está
   comprobado contra el historial.
2. **H05 → H06, H07** — precedencia respetada a propósito: H06 y H07 son los que ve el lector
   («4 comentarios», un hilo), pero ambos son comparaciones contra una columna que admite tres
   grafías. H05 va antes aunque por sí solo no rompa nada visible.
3. **H08, H09** — la feature no puede operarse: no hay forma de aprobar un comentario, y las dos
   reglas de autorización que existen no coinciden entre sí.
4. **H10, H11, H12** — integridad del árbol de comentarios. Son tres cuestiones independientes y
   ninguna bloquea a otra: H10 es validación de entrada, H11 es esquema, H12 es borrado en cascada.
   H11 es el menos reversible de los tres, por SQLite.
5. **H13, H14, H15, H16, H17** — contrato y validación. Radio limitado a quien integra o comenta,
   todos reversibles en código salvo los datos ya escritos.
6. **H18** — riesgo arquitectónico. No bloquea a nadie y no se le atribuye ninguna rotura; lo que
   sostiene su prioridad es que doce hallazgos están escritos dentro de ese bloque y que dos copias
   de la misma regla ya han divergido (H08, H15).
7. **H19, H20** — la red que debería haber cazado todo lo anterior. H20 detrás de H01 en valor.
8. **H21, H22, H23** — front y presentación. H21 detrás de H06 y H07 por dependencia real.

## Recuento de verificación

- **Hallazgos candidatos: 27.**
- **Hallazgos en la tabla: 23** (H01–H23; no hay H24).
- **Completamente verificados: 19** — H01, H02, H03, H05, H06, H07, H08, H09, H10, H11, H12, H13,
  H14, H15, H17, H18, H19, H20, H21. (En H19, el estado de la suite es evidencia de terceros, no mía.)
- **Parcialmente verificados: 4** — el patrón está observado en el código y la consecuencia no está
  reproducida:
  - **H04** — condición de carrera verificada por inspección, no reproducida con peticiones concurrentes.
  - **H16** — ausencia de límite observada, impacto real no medido.
  - **H22** — petición duplicada observada, coste no medido.
  - **H23** — construcción de la fecha observada, desfase no reproducido.
- **Descartados: 4.**
  - **H24 (retirado tras revisión)** — `key={index}` en `frontend/src/componentes/Comentarios.jsx:57`
    y `:66` está observado, pero el fallo visual que describía (React reutilizando nodos y cruzando
    autor y texto) no está demostrado para estos componentes, así que no se presenta como hallazgo.
  - XSS en el cuerpo del comentario — `Comentarios.jsx:61` y `:70` renderizan `{c.cuerpo}` como
    texto y React escapa.
  - Respuestas colgadas de un padre de otro post ya presentes en los datos — las consultas sobre
    `backend/tmp/db.sqlite3` devuelven 0 filas; solo queda el hueco de validación, que es H10.
  - Cuarta copia de la URL base de la API (`Comentarios.jsx:4`) — fuera de alcance: es transversal
    del front, no de esta rebanada.
- **Deliberadamente fuera de la tabla:** la ausencia de índice en `comentarios.post_id`. Está
  observada en el esquema, pero dentro de esta auditoría no puedo demostrar ningún efecto concreto
  —9 filas en la tabla y ninguna medición de latencia—, así que se queda como riesgo de rendimiento
  no medido y no como hallazgo.

## Parte B — Las tres líneas

1. El agente devolvió 27 hallazgos candidatos; comprobé completamente 19 abriendo los archivos y revisando la evidencia.

2. No supe decidir la prioridad de H04 frente a otros hallazgos de seguridad: el daño potencial es alto, pero la condición de carrera no fue reproducida ni medí su frecuencia.

3. No pude verificar el desfase de fecha descrito en H23: el patrón de UTC sin zona está en el código, pero no observé el problema en ejecución.
