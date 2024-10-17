# Integración de Mercado Pago con Suscripciones

En este documento vamos a aprender a agregar suscripciones a nuestra aplicación utilizando [Suscripciones sin plan asociado con pago pendiente](https://www.mercadopago.com.ar/developers/es/docs/subscriptions/integration-configuration/subscription-no-associated-plan/pending-payments). Los usuarios van a poder agregar mensajes a una lista de mensajes siempre y cuando estén suscritos.

Antes de continuar, asegurate de haber [configurado Mercado Pago](../configuracion-mercadopago/README.md) y haber [expuesto el puerto 3000 al exterior](../exponer-puerto/README.md).

## Indice

1. [Revisando nuestra aplicación](#revisando-nuestra-aplicación)
2. [Crear una suscripción](#crear-una-suscripción)
3. [Realizar una suscripción de prueba](#realizar-una-suscripción-de-prueba)
4. [Configurar webhook de notificaciones](#configurar-webhook-de-notificaciones)
5. [Recibir notificaciones](#recibir-notificaciones)
6. [Actualizar el estado de la suscripción](#actualizar-el-estado-de-la-suscripción)
7. [Probar la integración](#probar-la-integración)

## Revisando nuestra aplicación

En la página de inicio de nuestra aplicación (`/src/app/page.tsx`) se renderizan cosas diferentes dependiendo de si el usuario tiene o no una suscripción activa:
- Si el usuario tiene una suscripción activa: Se renderiza un formulario para agregar un mensaje a nuestra lista de mensajes.
- Si no tiene una suscripción activa: Se renderiza un formulario para suscribirse, que al hacer submit, se redirecciona al usuario a Mercado Pago para que pueda pagar.

```tsx
import {redirect} from "next/navigation";
import {revalidatePath} from "next/cache";

import api from "@/api";

export default async function SuscripcionesPage() {
  // Obtenemos los mensajes y el usuario
  const messages = await api.message.list();
  // Obtenemos el usuario
  const user = await api.user.fetch();

  async function suscribe(formData: FormData) {
    "use server";

    // Obtenemos el email del usuario
    const email = formData.get("email");

    // Suscribimos al usuario
    const url = await api.user.suscribe(user.id, email as string);

    // Redireccionamos al usuario a Mercado Pago para que pueda pagar
    redirect(url);
  }

  async function add(formData: FormData) {
    "use server";

    // Obtenemos el mensaje del usuario
    const message = formData.get("message");

    // Agregamos el mensaje a nuestra lista
    await api.message.add(message as string);

    // Revalidamos la página para que se muestren los mensajes actualizados
    revalidatePath("/");
  }

  return (
    <div className="grid gap-12">
      {user.suscription ? (
        <form action={add} className="grid gap-4">
          <textarea className="border border-blue-400 p-2" name="message" rows={4} />
          <button className="rounded-md bg-blue-400 px-4 py-2 text-white" type="submit">
            Submit
          </button>
        </form>
      ) : (
        <form action={suscribe} className="grid gap-4">
          <input
            className="border border-blue-400 p-2"
            defaultValue={user.email}
            name="email"
            placeholder="goncy@goncy.com"
            type="email"
          />
          <button className="rounded-md bg-blue-400 px-4 py-2 text-white" type="submit">
            Suscribirse
          </button>
        </form>
      )}
      <ul className="grid gap-4">
        {messages.map((message) => (
          <li key={message.id} className="bg-white/5 p-4">
            <p>{message.text}</p>
          </li>
        ))}
      </ul>
    </div>
  );
}
```

> [!NOTE]
> En esta aplicación de ejemplo siempre vamos a tener un usuario (que se obtiene desde `/db/user.db`), lo único que vamos a hacer es interactuar con la propiedad `suscription` del usuario. En una aplicación real vas a manejar la lógica de autenticación normalmente.

## Crear una suscripción

Dentro de `/src/api.ts`, la función `suscribe` en `user` se encarga de crear una suscripción sin plan asociado (con pago pendiente) y devolver el init point (url de pago):

```ts
const api = {
  user: {
    async suscribe(email: string) {
      const suscription = await new PreApproval(mercadopago).create({
        body: {
          back_url: process.env.APP_URL!,
          reason: "Suscripción a mensajes de muro",
          auto_recurring: {
            frequency: 1,
            frequency_type: "months",
            transaction_amount: 100,
            currency_id: "ARS",
          },
          payer_email: email,
          status: "pending",
        },
      });

      return suscription.init_point!;
    },
  }
}
```

Las suscripciones de pago pueden ser de 2 tipos:
- Suscripciones con plan asociado: Consta de 2 pasos, primero crear el plan, el cual estipula el título, monto, descripción, etc. Y el segundo paso es crear la suscripción. El problema es que para hacer esto, necesitamos un `card_token_id` el que generalmente obtenemos mediante Checkout Bricks o la Checkout API (la cual no vamos a implementar acá), queremos redirigir al usuario a Mercado Pago para que pueda pagar, similar a lo que haríamos con una preferencia de pago.
- Suscripciones sin plan asociado: Son suscripciones que se crean en el momento de pago, sin pasar por la creación de un plan. Al igual que las preferencias, nos devuelve un `init_point` para redirigir al usuario a Mercado Pago para completar el pago. Estas suscripciones pueden ser o de pago autorizado (necesitamos un `card_token_id`, lo cual no queremos ahora), o de pago pendiente para que el usuario complete el pago en Mercado Pago (justo lo que queremos).

Como te habrás dado cuenta, arriba estamos creando una suscripción sin plan asociado y de pago pendiente (status `pending`) y luego devolvemos el `init_point` y usamos esa URL para redirigir al usuario a Mercado Pago.

## Realizar una suscripción de prueba

Ahora vamos a iniciar sesión con nuestra cuenta de prueba comprador, vamos a ir a la página de inicio de nuestra aplicación y en el formulario de suscripción, vamos a escribir el email de nuestro usuario de la cuenta de prueba comprador de Mercado Pago (podés obtener este email entrando a [este link](https://www.mercadopago.com.ar/hub-engine/hubs/my-profile) desde el navegador donde estás logeado con tu cuenta de prueba de comprador de Mercado Pago). Eso nos va a redirigir a Mercado Pago. Completemos el pago y deberíamos ver algo como esto:

![image](./screenshots/suscripcion-aprobada.jpg)

Bien, nuestra suscripción fue aprobada. Ahora necesitamos que nuestra aplicación sea notificada de esta transacción para que pueda actualizar la suscripción y mostrar el formulario para agregar un mensaje.

> [!IMPORTANT]
> Es importante que uses el mail de la cuenta de prueba comprador de Mercado Pago, ya que si usas otro mail, posiblemente obtengas un error (poco descriptivo) y no puedas suscribirte.

## Configurar webhook de notificaciones

Vamos a ir a Mercado Pago y en la sección de `Webhooks` del panel izquierdo, vamos a `Configurar notificaciones`. Aseguratede estar en `Modo productivo` y en la URL de producción, pegamos la URL que tenemos en `APP_URL` dentro de nuestro archivo `.env.local` (más el endpoint `/api/mercadopago` al final) y seleccionamos `Planes y Suscripciones` en "Eventos".

![image](./screenshots/webhook-config.jpg)

Si vamos a `Simular notificación` y emitimos, deberíamos ver un mensaje similar a este indicando de que hubo un error (ya que la suscripción no existe) y también deberíamos ver un log en la terminal de nuestro equipo local hacia `/api/mercadopago` incluyendo información sobre la notificación.

![image](./screenshots/webhook-error.jpg)

Si bien no funcionó, nos sirve por que sabemos que Mercado Pago puede comunicarse con nuestra aplicación.

## Recibir notificaciones

Tenemos un Route Handler (`src/app/api/mercadopago/route.ts`) definido en nuestra aplicación que se encarga de recibir las notificaciones de Mercado Pago.

```ts
import {PreApproval} from "mercadopago";

import api, {mercadopago} from "@/api";

export async function POST(request: Request) {
  // Obtenemos el cuerpo de la petición que incluye el tipo de notificación
  const body: {data: {id: string}} = await request.json();

  // Obtenemos la suscripción
  const preapproval = await new PreApproval(mercadopago).get({id: body.data.id});

  // Si se aprueba, actualizamos el usuario con el id de la suscripción
  if (preapproval.status === "authorized") {
    // Actualizamos el usuario con el id de la suscripción
    await api.user.update({suscription: preapproval.id});
  }

  // Respondemos con un estado 200 para indicarle que la notificación fue recibida
  return new Response(null, {status: 200});
}
```

> [!NOTE]
> Es importante siempre retornar un estado 200 para indicarle a Mercado Pago que la notificación fue recibida. Solo debemos retornar un estado que no sea 200 cuando hubo algún error por el cual queremos que Mercado Pago nos notifique nuevamente.

## Actualizar el estado de la suscripción

Este Route Handler va a recibir las notificaciones de pago de Mercado Pago, va a obtener la suscripción usando el ID que nos llega en la notificación. En caso de que la suscripción haya sido autorizada, va a actualizar el usuario con el ID de la suscripción.

En una aplicación real deberíamos verificar la concordancia de la clave secreta, devolver errores más descriptivos, actualizar la suscripción si el usuario se da de baja y más, pero por simplicidad y tiempo te voy a dejar esa tarea a vos, podés ver más [acá](https://www.mercadopago.com.ar/developers/es/docs/your-integrations/notifications/webhooks#configuracinatravsdetusintegraciones).

## Probar la integración

Ahora vamos a intentar de hacer el flujo completo de suscribirnos y luego agregar un mensaje a nuestra lista de mensajes y veamos si funciona.

![image](./screenshots/suscripcion-aprobada.jpg)

Excelente, nuestra suscripción fue aprobada, la notificación fue recibida y nuestro mensaje fue agregado a la lista ✨.

> [!NOTE]
> Si querés seguir probando este flujo, podés modificar el usuario desde `/db/user.db` y poner la suscripción en `null` para que puedas volver a suscribirte.

---

[Volver al inicio](../README.md)