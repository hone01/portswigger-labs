# CSRF where token is duplicated in cookie

**Plataforma:** PortSwigger Web Security Academy
**Dificultad:** Practitioner
**Categoría:** CSRF · CRLF injection (encadenadas)

---

## Objetivo

La funcionalidad de cambio de email de la aplicación es vulnerable a CSRF. La app intenta protegerse con la técnica insegura de "double submit" cookie. El objetivo es alojar en el exploit server una página HTML que, al ser visitada por la víctima, le cambie la dirección de email.

Credenciales de prueba: `wiener:peter`

---

## La vulnerabilidad

Aquí se encadenan **dos** fallos:

**1. Double-submit cookie mal implementado (CSRF).**
La app valida el token CSRF comparando el parámetro `csrf` del cuerpo de la petición con una cookie llamada `csrf`. Si ambos coinciden, da la petición por legítima. El problema: el token **no está atado a la sesión del usuario**. A la app le da igual *qué* valor tenga el token — solo comprueba que el del cuerpo y el de la cookie sean iguales. Si un atacante consigue fijar ambos con un valor que él controla, la protección se rompe.

**2. CRLF injection (el vector para fijar la cookie).**
El endpoint de búsqueda refleja el parámetro `search` sin sanitizar dentro de una cabecera `Set-Cookie` de la respuesta (`LastSearchTerm=...`). Como no filtra los caracteres CRLF (`\r\n`, codificados como `%0d%0a`), es posible inyectar un salto de línea y con él una **cabecera `Set-Cookie` adicional arbitraria**. Esto permite plantar en el navegador de la víctima una cookie `csrf` con el valor que quiera el atacante.

El concepto clave: `\r\n` es el **delimitador estructural** que separa cabeceras en HTTP. Meterlo en un dato que se refleja en una cabecera permite "escapar" del dato e inyectar estructura nueva — pasar de dato a cabecera.

---

## Pasos

1. **Confirmar la CSRF básica.** Interceptar el cambio de email y ver que la petición depende de un token `csrf` que se compara contra una cookie del mismo nombre.

2. **Encontrar la CRLF injection.** En el endpoint de búsqueda, el parámetro `search` se refleja dentro de la cabecera `Set-Cookie: LastSearchTerm=...`. Probar en Repeater un payload con CRLF codificado:

   ```
   /?search=test%0d%0aSet-Cookie:%20csrf=arbitrario%3b%20SameSite=None
   ```

   En la respuesta aparecen **dos** cabeceras `Set-Cookie`: la legítima (`LastSearchTerm`) y la inyectada (`csrf=arbitrario`). Inyección confirmada.

   > `SameSite=None` es imprescindible: sin él, la cookie no viaja en un contexto cross-site y el ataque no llega a ejecutarse cuando la víctima abre la página del atacante.

3. **Construir el exploit HTML.** Dos piezas juntas:
   - Un `<img>` que apunta a la URL de búsqueda con el payload CRLF → planta la cookie `csrf` en el navegador de la víctima al cargar (usando el evento de error de la imagen para encadenar el envío).
   - Un `<form>` oculto que se autoenvía al endpoint de cambio de email, con el campo `csrf` puesto al mismo valor arbitrario que la cookie inyectada.

   Como cookie y campo llevan el mismo valor, la validación double-submit pasa, y el email se cambia sin consentimiento de la víctima.

4. **Subir al exploit server y entregarlo a la víctima.** Lab resuelto.

---

## Causa raíz

Dos raíces, una por cada fallo encadenado:

- **CSRF:** el token no está vinculado a la sesión del usuario. La validación solo comprueba que dos valores coincidan, no que el token pertenezca de verdad a esa sesión — así que un valor arbitrario controlado por el atacante pasa el control.
- **CRLF injection:** entrada de usuario (`search`) reflejada en una cabecera de respuesta sin sanitizar los caracteres de control `\r\n`.

---

## Remediación

- **Atar el token CSRF a la sesión** del usuario y validar esa correspondencia server-side, en vez de la comparación ciega double-submit. Un token que no pertenece a la sesión debe rechazarse aunque coincida con la cookie.
- **Sanitizar la entrada** que se refleja en cabeceras: eliminar o codificar los caracteres CRLF (`\r\n`) antes de incluir cualquier dato controlado por el usuario en una cabecera de respuesta.
- Defensa en profundidad: `SameSite=Strict` o `Lax` en las cookies de sesión reduce la superficie de ataques cross-site.

---

## Qué aprendí

- El CRLF (`\r\n`) no es un carácter cualquiera: es el delimitador que da estructura al protocolo HTTP. Inyectarlo permite escapar del contexto de dato al de estructura.
- Una protección CSRF puede estar "presente" y aun así ser inútil si el token no se ata a la sesión.
- Construir el exploit HTML a mano (en vez de con un generador automático) obliga a entender *por qué* funciona: la cookie plantada y el campo del formulario tienen que llevar el mismo valor para que el double-submit pase.
