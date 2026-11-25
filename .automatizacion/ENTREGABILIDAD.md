# Entregabilidad del correo de LIDATO · pendientes

Estado al 25-nov-2026. Origen: el aviso del panel diario salió tres veces con
estado `delivered` y la destinataria no vio ninguno. `delivered` solo significa
que el servidor del destinatario aceptó el mensaje; el correo entró a INCOMELEC
y quedó retenido de su lado.

Las tres acciones van en este orden. La 1 y la 2 son las que resuelven.

---

## 1 · Publicar el registro DMARC de `lidato.site`

Hoy el dominio tiene SPF y DKIM verificados en Resend, pero **no tiene DMARC**.
Es el hueco más grande: un dominio recién creado y sin DMARC es el perfil que
los filtros corporativos mandan a cuarentena.

Dónde: en el DNS de `lidato.site` (Cloudflare → el dominio → DNS → Records →
Add record). No es en Workers & Pages.

```
Tipo:   TXT
Nombre: _dmarc
Valor:  v=DMARC1; p=none; rua=mailto:operaciones@lidato.site;
TTL:    Auto
Proxy:  desactivado (los TXT no se proxean)
```

Cloudflare agrega el dominio solo, así que el registro queda como
`_dmarc.lidato.site`.

`p=none` es a propósito: solo observa y reporta, no bloquea nada. La
documentación de Resend dice empezar ahí y subir a `p=quarantine` y luego a
`p=reject` únicamente cuando se confirme que todo el correo legítimo pasa DMARC.
Subirlo antes de tiempo puede tumbar correo bueno.

Para comprobarlo, una vez publicado:

```
dig TXT _dmarc.lidato.site +short
```

---

## 2 · Pedir a TI de INCOMELEC que permita el dominio

Es lo único que garantiza la bandeja de entrada en un correo corporativo.
Texto para enviarles tal cual:

> Buen día,
>
> Les escribo porque los correos automáticos que enviamos al área de tesorería
> no están llegando a la bandeja de entrada. Nuestro proveedor confirma que el
> servidor de INCOMELEC los aceptó, así que entendemos que están quedando
> retenidos en el filtro o en cuarentena.
>
> ¿Podrían revisar la cuarentena y agregar nuestro dominio de envío a la lista
> de permitidos?
>
> - Dominio de envío: lidato.site
> - Remitente: operaciones@lidato.site
> - Los mensajes salen autenticados con SPF y DKIM
>
> Les paso el Message-ID de uno de los correos retenidos, por si les sirve para
> el rastreo de mensajes:
> 010301a213042089-2229328c-6ed1-4c5d-9570-d95d43dbb8e1-000000@sa-east-1.amazonses.com
>
> No pedimos que autoricen direcciones IP: las de nuestro proveedor cambian, y
> permitir el dominio autenticado es más seguro y estable.
>
> Gracias.

Si usan Microsoft 365, la cuarentena se revisa en
`security.microsoft.com/quarantine`. Liberar un mensaje libera **ese** mensaje;
para los siguientes hace falta la lista de permitidos. Lo clasificado como
phishing de alta confianza solo lo libera un administrador.

---

## 3 · Que la destinataria responda una vez el correo

Una respuesta desde su buzón es la señal más fuerte que existe para su propio
filtro. Basta con que conteste «recibido» una vez.

---

## 4 · Webhooks (opcional, para enterarnos sin que el cliente reclame)

La cuenta de Resend no tiene ningún webhook configurado. Con un endpoint que
escuche estos eventos sabríamos del problema el mismo día:

- `email.bounced` — rechazo permanente
- `email.complained` — lo marcaron como spam
- `email.delivery_delayed` — aplazado, reintenta hasta 14 h
- `email.suppressed` — no se envió, dirección suprimida

Deduplicar por la cabecera `svix-id`, que se repite en los reintentos.

Se puede montar como un Worker de Cloudflare que escriba a una base. Requiere
desplegar código, cosa que el conector actual de Cloudflare no permite hacer
desde una corrida automática.

---

## Lo que NO hay que hacer

- **No reenviar el mismo correo** cuando figura `delivered`: cae exactamente en
  el mismo sitio. Se probó tres veces el 25-nov.
- **No volver a Gmail.** Se probó de tres formas y el conector reescribe las
  direcciones y borra las imágenes externas.
- **No mover los envíos a otro dominio** para esquivar el filtro: empeora la
  reputación de los dos.

---

Fuentes: documentación de Resend consultada el 25-nov-2026 —
`resend.com/guides/email/users-never-got-login-code`,
`resend.com/docs/dashboard/domains/dmarc`,
`resend.com/docs/webhooks/event-types`,
`resend.com/guides/email/new-domain-emails-going-to-spam`.
