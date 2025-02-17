---
layout: ~/layouts/MainLayout.astro
title: Renderizado en el servidor
i18nReady: true
---

**Renderizado en el servidor**, también conocido como SSR (server side rendering), se puede habilitar en Astro. Cuando habilitas SSR puedes:

- Implementar sesiones para iniciar sesión en su aplicación.
- Renderizar datos desde una llamada de API dinámicamente con `fetch`.
- Desplegar su sitio en un servidor usando un *adaptador*.

## Habilitando SSR en su proyecto

Para habilitar SSR se necesita usar un adaptador. Esto se debe a que SSR requiere un _entorno de ejecución_ de servidor: el ambiente donde se ejecutará su código en el servidor. Este entorno de ejecución proporciona una API que su código en el servidor puede usar.

La instalación de un adaptador le da a Astro acceso a la API correspondiente y le permite generar un script que ejecuta su proyecto en ese tipo de servidor.

Los siguientes adaptadores están disponibles hoy y habrán muchos más en el futuro:

- [Cloudflare](/es/guides/integrations-guide/cloudflare/)
- [Deno](/es/guides/integrations-guide/deno/)
- [Netlify](/es/guides/integrations-guide/netlify/)
- [Node.js](/es/guides/integrations-guide/node/)
- [Vercel](/es/guides/integrations-guide/vercel/)

Puedes encontrar las instrucciones de configuración en los enlaces anteriores de acuerdo al adaptador. Para completar los siguientes dos pasos (usaremos `my-adapter` como ejemplo) para habilitar SSR.

1. Instala el adaptador a las dependencias de tu proyecto a través de npm o el administrador de paquetes de tu elección.

    ```bash
    npm install --save-dev @astrojs/my-adapter
    ```

2. [Agrega el adaptador](/es/reference/configuration-reference/) a tu archivo de configuración `astro.config.mjs` de la siguiente forma. 

    ```js ins={3,6-7}
    // astro.config.mjs
    import { defineConfig } from 'astro/config';
    import myAdapter from '@astrojs/my-adapter';

    export default defineConfig({
      output: 'server',
      adapter: myAdapter(),
    });
    ```

## Características

Astro seguirá siendo un generador de sitios estáticos de forma predeterminada. Pero una vez que habilites un adaptador de renderizado en el servidor, **cada ruta en la carpeta de páginas se convertirá en una ruta renderizada por el servidor** y algunas características nuevas estarán disponibles para ti.

### `Astro.request.headers`

Los encabezados de la solicitud están disponibles en `Astro.request.headers`. Es un objeto [Headers](https://developer.mozilla.org/en-US/docs/Web/API/Headers), es un objeto similar a un Map donde puedes recuperar encabezados como la cookie.

```astro title="src/pages/index.astro" {2}
---
const cookie = Astro.request.headers.get('cookie');
// ...
---
<html>
  <!-- Maquetado aquí... -->
</html>
```

### `Astro.redirect`

En el objeto `Astro` global, este método te permite redirigir a otra página. Puedes hacer esto después de verificar si el usuario ha iniciado sesión obteniendo la sesión desde una cookie.

```astro title="src/pages/account.astro" {8}
---
import { isLoggedIn } from '../utils';

const cookie = Astro.request.headers.get('cookie');

// Si el usuario no ha iniciado sesión, redirígelo a la página de inicio de sesión.
if (!isLoggedIn(cookie)) {
  return Astro.redirect('/login');
}
---
<html>
  <!-- Maquetado aquí... -->
</html>
```

### `Response`

También puedes devolver una [Response](https://developer.mozilla.org/es/docs/Web/API/Response) desde cualquier página. Puedes hacer esto para devolver un 404 en una página dinámica luego de buscar y no encontrar la id en la base de datos.

```astro title="src/pages/[id].astro" {8-11}
---
import { getProduct } from '../api';

const product = await getProduct(Astro.params.id);

// Producto no encontrado
if (!product) {
  return new Response(null, {
    status: 404,
    statusText: 'Not found'
  });
}
---
<html>
  <!-- Maquetado aquí... -->
</html>
```

## Rutas API

Una [ruta API](https://medium.com/@rajat_m/what-are-restful-routes-and-how-to-use-them-929129ae7bf6) es un archivo `.js` o `.ts` dentro de la carpeta `src/pages/` que recibe una [Request](https://developer.mozilla.org/es/docs/Web/API/Request) y devuelve una [Response](https://developer.mozilla.org/en-US/docs/Web/API/Response). Una característica poderosa de SSR: Las rutas API permiten ejecutar código del lado del servidor de forma segura.

### SSR y Rutas

En Astro, estas rutas se convierten en rutas renderizadas por el servidor, lo que te permite utilizar características que no estaban disponibles en el cliente o que requieren llamadas explícitas a un servidor y código extra en el cliente para renderizar el resultado.

En el siguiente ejemplo, se utiliza una ruta API para buscar un producto en una base de datos, sin la necesidad de generar una página por cada una de los productos disponibles.

```js title="src/pages/[id].js"
import { getProduct } from '../db';

export async function get({ params }) {
  const { id } = params;
  const product = await getProduct(id);

  if (!product) {
    return new Response(null, {
      status: 404,
      statusText: 'Not found'
    });
  }

  return new Response(JSON.stringify(product), {
    status: 200
  });
}
```

En este ejemplo, se puede retornar código HTML válido para renderizar la página entera o parte del contenido.


Además de consultar contenido y renderizado del lado del servidor, las rutas API pueden ser usadas como endpoints de una API REST para correr funciones tales como autenticación, acceder a bases de datos y realizar verificaciones sin exponer datos sensibles al cliente.

En el siguiente ejemplo se utiliza una ruta API para verificar el reCaptcha v3 de Google sin exponer los secretos al cliente.


```astro title="src/pages/index.astro"
<html>
  <head>
    <script src="https://www.google.com/recaptcha/api.js"></script>
  </head>

  <body>
    <button class="g-recaptcha" 
      data-sitekey="PUBLIC_SITE_KEY" 
      data-callback="onSubmit" 
      data-action="submit"> ¡Haz click para verificar el captcha! </button>

    <script is:inline>
      function onSubmit(token) {
        fetch("/recaptcha", {
          method: "POST",
          body: JSON.stringify({ recaptcha: token })
        })
        .then((response) => response.json())
        .then((gResponse) => {
          if (gResponse.success) {
            // La verificación del Captcha fue positiva
          } else {
            // Falló la verificación del Captcha
          }
        })
      }
    </script>
  </body>
</html>
```

En la ruta API puedes definir valores secretos o leer tus variables de entorno secretas.

```js title="src/pages/recaptcha.js"
import fetch from 'node-fetch';

export async function post({ request }) {
  const data = request.json();

  const recaptchaURL = 'https://www.google.com/recaptcha/api/siteverify';
  const requestBody = {
    secret: "YOUR_SITE_SECRET_KEY",   // Esta puede ser una variable de entorno
    response: data.recaptcha          // El token recibido desde el cliente
  };

  const response = await fetch(recaptchaURL, {
    method: "POST",
    body: JSON.stringify(requestBody)
  });

  const responseData = await response.json();

  return new Response(JSON.stringify(responseData), { status: 200 });
}
```
