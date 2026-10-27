# LAB-01 

**Estudiante:** Luis Enrique Millan Inzunza


## Observaciones del entorno

- El servidor de pgAdmin es compartido y los roles son globales. El rol `analista` ya existía, asi que para no reutilizar permisos de otro usuario creé `analista_tunombre`. (Si también cambiaste `soporte`, agrégalo aquí.)
- El `GRANT ... WITH SET TRUE` dio error de sintaxis, por lo que usé `GRANT rol TO CURRENT_USER;` para poder probar con `SET ROLE`.
- Versión de PostgreSQL del servidor: *(pega aquí el resultado de `SELECT version();`)*.

---

## Clasificación

Analicé los nueve campos de la tabla `clientes`. Todos requieren algún nivel de protección, aunque por razones distintas.

| Campo | ¿Identifica directamente? | ¿Puede contribuir a identificar? | ¿Requiere protección? | Justificación |
|---|---|---|---|---|
| id | No  | si | si | Es una llave interna que enlaza al cliente con todos sus datos y con el mapa de correspondencia. |
| nombre | si | si | si | Identificador directo más evidente. |
| correo | si | si | si | Normalmente contiene el nombre y es único. También sirve para phishing o suplantación. |
| telefono | si | si | si | Dato único que permite contactar o localizar a la persona. |
| rfc | si | Si | Si | Identificador fiscal único construido con datos del titular. Su exposición facilita fraude. |
| edad | No | si | Si | Combinada con ciudad y código postal distingue a una persona en un grupo pequeño. |
| ciudad | No | si | si | Reduce el universo de personas posibles. |
| codigo_postal | No | si | si | Ubica una zona más específica que la ciudad. Es un cuasi-identificador fuerte. |
| fecha_alta | No | si  | Si  | Revela cuándo se hizo cliente y puede ayudar a distinguir registros. Además sirve para la retención. |

**¿Qué campos identifican directamente a una persona?**
El nombre, el correo, el teléfono y el RFC. El `id` no identifica directamente, pero enlaza todos los datos del cliente.

**¿Qué campos no identifican por si solos pero podrían contribuir a hacerlo?**
La edad, la ciudad, el código postal y la fecha de alta. También el `id` si se expone.

**¿Qué combinación de campos podría incrementar el riesgo de reidentificación?**
Edad + ciudad + código postal, y más si se agrega la fecha de alta. Por ejemplo, solo hay un cliente de 41 años en Culiacán con código postal 80000, asi que esos tres datos apuntan a una sola persona aunque se quite el nombre.

**¿Todos los datos que deben protegerse son necesariamente datos personales sensibles desde el punto de vista jurídico?**
No. Una cosa es que un dato necesite protección y otra que la ley lo considere sensible. Los sensibles tienen que ver con la parte más íntima de la persona (salud, origen étnico, creencias, ideología, preferencia sexual), porque su divulgación puede causar discriminación o un daño serio. El nombre, correo, teléfono y RFC son datos personales que si hay que proteger, pero no son sensibles en ese sentido. La edad, la ciudad y el código postal tampoco lo son, pero se cuidan porque combinados pueden reidentificar a alguien. Lo que determina la protección es el riesgo si se expone.

---

## Enmascaramiento

**¿Qué se ocultó?**
Parte del correo, del teléfono y del RFC. Por ejemplo, el correo muestra solo las dos primeras letras y el dominio, el teléfono solo los últimos 4 dígitos y el RFC solo los últimos 4 caracteres.

**¿Para quién?**
Para el rol `soporte`, que necesita identificar al cliente para atenderlo pero no requiere ver todos sus datos completos.

**¿Por qué?**
Por el principio de mínimo privilegio: soporte solo debe ver lo necesario para su función. Asi se reduce el riesgo de exposición o mal uso de datos personales.

**¿Qué mecanismo se utilizó?**
Una vista (`v_clientes_soporte`) que aplica el enmascaramiento con `CONCAT`, `LEFT`, `RIGHT` y `SUBSTRING`. Al rol `soporte` se le dio `USAGE` sobre el esquema `public` y `SELECT` solo sobre la vista, y se ejecutó `REVOKE ALL` sobre la tabla `clientes`. Es un enmascaramiento dinámico: el dato original no se modifica, solo cambia lo que se muestra.

---

## Seudonimización

**¿Cómo se generaron los tokens?**
Con `gen_random_uuid()`, que crea un UUID aleatorio para cada cliente. Como el token es aleatorio, no se puede deducir el `cliente_id` a partir de él.

**¿Dónde se encuentra el mapa de correspondencia?**
En la tabla `pii.mapa_cliente`, dentro del esquema `pii`, separado del esquema `analitica` donde está el conjunto de datos de análisis (`analitica.clientes`).

**¿Quién debería poder acceder a él?**
Solo personal autorizado y con una necesidad justificada de reidentificar, como el DBA o un responsable de datos. El rol de analítica no debe tener acceso.

**¿Por qué se separaron los permisos?**
Porque si el mismo usuario pudiera consultar los datos seudonimizados y el mapa, podría hacer un `JOIN` por el token y recuperar la identidad, y la seudonimización no serviría de nada. Se aplican separación de funciones y mínimo privilegio. El rol `analista_tunombre` solo tiene `USAGE` sobre el esquema `analitica` y `SELECT` sobre `analitica.clientes`.

---

## Anonimización

**¿Qué identificadores fueron eliminados?**
Nombre, correo, teléfono, RFC e `id`. Las consultas agregadas solo muestran rango de edad, región y conteo.

**¿Qué atributos fueron generalizados?**
La edad, convertida en rangos (`18-30`, `31-45`, `46+`), y el código postal, reducido a sus dos primeros dígitos como región. En el reto amplié los rangos de edad y reduje la región a un dígito.

**¿Qué grupos fueron suprimidos?**
Con `HAVING COUNT(*) >= 5`  todos los grupos fueron suprimidos, porque solo hay 5 clientes y ningún grupo llega al umbral. En el reto, con umbral 3, se suprimió el grupo `41+`, que tiene 2 clientes.

**¿Qué riesgo de reidentificación permanece?**
Aunque se eliminaron los identificadores directos, siguen existiendo cuasi-identificadores. Con una población tan pequeña, y si alguien conoce a una persona del conjunto, podría deducir a qué grupo pertenece. Por eso hay que combinar la anonimización con control de acceso.

---

## Retención

**¿Qué criterio se utilizó?**
Un periodo de cinco años a partir de `fecha_alta`, identificado con:

```sql
SELECT *
FROM clientes
WHERE fecha_alta < CURRENT_DATE - INTERVAL '5 years';
```

Con los datos actuales devuelve 0 filas, porque todos los clientes se dieron de alta hoy. Para demostrar que la lógica funciona usé una consulta de demostración (`fecha_alta < CURRENT_DATE + INTERVAL '1 day'`), que devuelve los 5 registros.

**¿Qué información debería conservarse?**
La que tenga una finalidad vigente: clientes con relación comercial activa, y la información que deba conservarse por obligaciones fiscales, contractuales o legales, o por una investigación en curso.

**¿Qué información podría eliminarse?**
Los datos personales de clientes cuya fecha de alta supera los cinco años y que ya no tienen finalidad, obligación legal ni relación vigente. Mientras más información innecesaria se conserve, mayor es el impacto de una fuga.

**¿Qué controles deberían existir antes de eliminarla?**

- **Aprobación formal por escrito** del responsable del área, de legal o cumplimiento, y del responsable de seguridad o DBA.
- **Respaldo completo y verificado**, guardado con acceso restringido, y una prueba de que se puede restaurar. Conviene exportar aparte los registros a eliminar.
- **Evidencia:** consulta usada y resultado, aprobación, confirmación del respaldo, registro de la ejecución (fecha, usuario y filas eliminadas) y comprobación posterior. 
- **Procedimiento de recuperación ante borrado accidental:** detener nuevas eliminaciones, identificar qué se borró y cuándo, restaurar el respaldo en una base temporal, reinsertar solo los registros afectados, verificar y documentar el incidente. 
---

## Conclusiones

**¿Qué técnica utilizarías para un área de soporte?**
Enmascaramiento mediante una vista, junto con control de acceso. Soporte necesita identificar al cliente, asi que conserva el nombre, pero los datos más delicados (correo, teléfono, RFC) se muestran parcialmente. Se le da acceso solo a la vista y no a la tabla.

**¿Qué técnica utilizarías para un área de analítica?**
Seudonimización con separación de funciones. Analítica no necesita saber quién es cada cliente, solo analizar patrones, asi que trabaja con tokens y sin acceso al mapa de correspondencia.

**¿Qué técnica utilizarías para publicar estadísticas?**
Anonimización con agregación y generalización: rangos de edad, regiones amplias y supresión de grupos pequeños. Se publican solo resúmenes y no registros individuales, y se evalúa el riesgo de reidentificación.

**¿Por qué ninguna de estas técnicas sustituye al control de acceso?**
Porque cada una reduce el riesgo de una manera distinta, pero ninguna impide por si sola que alguien con permisos de más llegue a la tabla original o al mapa. El enmascaramiento solo cambia la visualización, la seudonimización se puede revertir con el mapa y la anonimización no garantiza anonimato total.
---

## Preguntas de reflexión

**1. ¿Cuál es la diferencia entre enmascaramiento y seudonimización?**
El enmascaramiento oculta cómo se muestra un dato (por ejemplo `j***@example.com`) sin cambiar lo almacenado. La seudonimización sustituye la identidad directa por un token y guarda aparte la información para volver a relacionarlo. El primero busca ocultar la visualización y el segundo sustituir la identidad.

**2. ¿Por qué la seudonimización no significa anonimización?**
Porque la relación entre el token y la persona sigue existiendo en el mapa de correspondencia, por lo que se puede revertir. Un dato seudonimizado sigue siendo un dato personal.

**3. ¿Por qué eliminar nombre y RFC no garantiza que un conjunto de datos sea anónimo?**
Porque quedan cuasi-identificadores como edad, ciudad, código postal y fecha de alta. Combinados, y cruzados con otras fuentes, pueden identificar a una persona, sobre todo en un conjunto pequeño.

**4. ¿Qué función tienen los cuasi-identificadores?**
Por si solos no identifican, pero combinados distinguen a una persona dentro de un grupo. Por eso la anonimización debe analizarlos y no solo quitar columnas con nombres evidentes.

**5. ¿Por qué es importante separar el mapa de seudonimización?**
Porque es la llave que permite reidentificar. Si quien analiza los datos también puede consultar el mapa, la seudonimización pierde su utilidad. Por eso debe tener controles más estrictos que los datos de análisis.

**6. ¿Qué principio de seguridad se aplica al impedir que el analista acceda al mapa?**
Separación de funciones y mínimo privilegio: el analista solo recibe los permisos que necesita para analizar.

**7. ¿Por qué una vista puede ser preferible a entregar acceso directo a la tabla?**
Porque permite mostrar solo las columnas necesarias, y en este caso enmascaradas, sin dar acceso a la tabla original. Asi el usuario no puede ver el resto de los datos ni los valores completos.

**8. ¿Qué diferencia existe entre proteger un dato para visualización y protegerlo contra acceso no autorizado?**
Proteger para visualización cambia lo que se ve (enmascaramiento), pero el dato original puede seguir accesible si hay permisos. Protegerlo contra acceso no autorizado significa impedir que quien no debe llegue al dato, con roles, `GRANT`, `REVOKE` y separación de esquemas.

**9. ¿Por qué la retención forma parte de la protección de datos?**
Porque un dato que ya no se conserva no puede filtrarse. Definir cuánto tiempo se guarda cada dato, según su finalidad y las obligaciones aplicables, reduce la exposición y respeta la minimización.

**10. ¿Qué riesgo existe si una organización conserva información indefinidamente?**
Aumenta el impacto de una fuga o incidente, porque hay más datos expuestos, muchos sin finalidad. También se incumple la minimización y se complica la administración, el respaldo y el cumplimiento normativo.
