---
title: Paginación
slug: pagination
date: 0012/01/01
number: 12
points: 10
photoUrl: http://www.flickr.com/photos/ikewinski/8625379401/
photoAuthor: Mike Lewinski
contents: Aprenderemos más sobre las suscripciones y cómo usarlas para controlar los datos.|Implementaremos la paginación de estilo infinito|Usaremos el paquete iron-router-progress para implementar una bonita barra de progreso al estilo iOS.|Crearemos una suscripción especial para tratar con los enlaces a las páginas de posts.
---

Nuestra aplicación va tomando forma y podemos esperar una gran éxito cuando salga al mercado.

Así que quizás debamos pensar un poco sobre cómo afectará al rendimiento el gran número de nuevos posts que vamos a recibir.

Hemos visto antes cómo una colección en el cliente pude contener un subconjunto de los datos en el servidor y lo hemos usado para nuestras notificaciones y comentarios.

Ahora pensemos en que todavía estamos publicando todos nuestros posts de una sola vez a todos los usuarios conectados. Si se publicaran miles de enlaces, esto sería un problema. Para solucionarlo tenemos que paginar nuestros posts.are posted, this will become problematic. To solve this, we need to paginate our posts.

### Añadiendo unos cuantos posts

En primer lugar, vamos a cargar los suficientes posts para que la paginación tenga sentido:

~~~js
// Fixture data 
if (Posts.find().count() === 0) {

  //...
  
  Posts.insert({
    title: 'The Meteor Book',
    userId: tom._id,
    author: tom.profile.name,
    url: 'http://themeteorbook.com',
    submitted: now - 12 * 3600 * 1000,
    commentsCount: 0
  });
  
  for (var i = 0; i < 10; i++) {
    Posts.insert({
      title: 'Test post #' + i,
      author: sacha.profile.name,
      userId: sacha._id,
      url: 'http://google.com/?q=test-' + i,
      submitted: now - i * 3600 * 1000,
      commentsCount: 0
    });
  }
}
~~~
<%= caption "server/fixtures.js" %>
<%= highlight "15~24" %>

Después de ejecutar `meteor reset` deberíamos ver algo como esto:

<%= screenshot "12-1", "Mostrando un montón de datos." %>

<%= commit "12-1", "Añadidos suficientes posts para hacer necesaria la paginación." %>

### Paginación infinita

Vamos a implementar una paginación de estilo estilo "infinito". Lo que queremos decir con esto es que primero mostramos, por ejemplo, 10 posts, con un enlace de "cargar más" en la parte inferior. Al hacer clic en este enlace se cargarán 10 más, y así _hasta el infinito y más allá_. Esto significa que podemos controlar todo nuestro sistema de paginación con un solo parámetro que representa el número de posts que mostraremos en pantalla.

Vamos a necesitar entonces una forma de pasar este parámetro al servidor para que sepa la cantidad de mensajes que debe enviar al cliente. Se da la circunstancia de que ya estamos suscritos a la publicación `posts` en el router, así que vamos a aprovecharlo y a dejar que el router maneje también la paginación.

La forma más fácil de configurar esto es hacer que el parámetro límite forme de la ruta, quedando las URL de la forma `http://localhost:3000/25`. Una ventaja añadida de utilizar la URL en vez de otros métodos es que si estamos viendo 25 posts y resulta que se recarga la página por error, todavía seguiremos viendo 25 posts.

Para hacer esto correctamente, tenemos que cambiar la forma en que nos suscribimos a los posts. Al igual que hicimos en el capítulo _Comentarios_, tendremos que mover nuestro código de suscripción desde el nivel de router al nivel de ruta.

Parece demasiado para hacerlo todo de una sola vez, pero se verá más claro escribiendo el código.

En primer lugar, vamos a dejar de suscribirnos a la publicación `posts` en el bloque `Router.configure()`. Simplemente elimina `Meteor.subscribe('posts')`, dejando sólo la suscripción `notifications`:

~~~js
Router.configure({
  layoutTemplate: 'layout',
  loadingTemplate: 'loading',
  waitOn: function() { 
    return [Meteor.subscribe('notifications')]
  }
});
~~~
<%= caption "lib/router.js" %>
<%= highlight "5" %>

A continuación, añadiremos el parámetro `postsLimit` al path de la ruta. Si añadimos un `?` después del parámetro, lo hacemos opcional. De forma que  nuestra ruta no sólo coincidirá con `http://localhost:3000/50`, sino también con `http://localhost:3000`.

~~~js
Router.map(function() {
  //...
  
  this.route('postsList', {
    path: '/:postsLimit?'
  });
});
~~~
<%= caption "lib/router.js" %>
<%= highlight "5" %>

Es importante señalar que un path de la forma `/:parameter?` coincide con todos los path posibles. Dado que cada ruta se analiza en orden sucesivo para comprobar si coincide con la ruta actual, tenemos que asegurarnos de que organizamos bien nuestras rutas con el fin de disminuir la especificidad.

En otras palabras, las rutas más específicas como `/posts/:_id` deben ir primero en código, y nuestra ruta `postsList` debería ir en la parte inferior del archivo, ya que más o menos coincide con todo.

Es el momento de abordar el difícil problema de suscribirse y encontrar los datos correctos. Tenemos que lidiar con el caso en el que el parámetro `postsLimit` no está presente, por lo que vamos a asignarle un valor predeterminado. Usaremos "5", que nos dará suficiente espacio para jugar con la paginación.

~~~js
Router.map(function() {
  //..
  
  this.route('postsList', {
    path: '/:postsLimit?',
    waitOn: function() {
      var postsLimit = parseInt(this.params.postsLimit) || 5; 
      return Meteor.subscribe('posts', {sort: {submitted: -1}, limit: postsLimit});
    }
  });
});
~~~
<%= caption "lib/router.js" %>
<%= highlight "6~9" %>

Yte habrás dado cuenta de que estamos pasando un objeto JavaScript `({limit: postsLimit})` junto con el nombre de nuestra publicación `posts`. Este objeto servirá como parámetro `options` en la llamada a `Posts.find()` en el lado del servidor. Vamos a cambiar a nuestro código en el servidor para implementarlo:

~~~js
Meteor.publish('posts', function(options) {
  return Posts.find({}, options);
});

Meteor.publish('comments', function(postId) {
  return Comments.find({postId: postId});
});

Meteor.publish('notifications', function() {
  return Notifications.find({userId: this.userId});
});
~~~
<%= caption "server/publications.js" %>
<%= highlight "1~3" %>

<% note do %>

### Paso de parámetros

Nuestro código de publicaciones está diciendo al servidor que puede confiar en cualquier objeto JavaScript enviado por el cliente (en nuestro caso, `{limit: postsLimit})` para servir como opciones para `find()`. Esto hace posible que los usuarios envíen cualquier opción a través de la consola del navegador.

En nuestro caso, esto es relativamente inofensivo, ya que todo lo que un usuario podría hacer es reordenar los mensajes de manera diferente, o cambiar el límite (que es lo que queremos hacer).

Pero no debes utilizar este modelo para almacenar datos privados en campos no publicados, ya que el usuario puede manipular los campos para acceder a ellos. Además también deberías evitar usarlo para el argumento de selección del `find()` por las mismas razones de seguridad.

Un patrón más seguro para asegurarnos el control de nuestros datos podría ser pasar los parámetros de forma individual en lugar de todo el objeto:

~~~js
Meteor.publish('posts', function(sort, limit) {
  return Posts.find({}, {sort: sort, limit: limit});
});
~~~

<% end %>

Ahora que nos suscribimos a nivel de ruta, tiene sentido establecer el contexto de datos en ese mismo lugar. Vamos a desviarnos un poco de nuestro patrón anterior y hace que la función `data` datos devuelva un objeto JavaScript en lugar de simplemente devolver un cursor. Esto nos permite crear un contexto de datos con nombre que llamaremos `posts`.

Lo que significa es que en lugar de disponer implícitamente de los datos en `this` dentro de la plantilla, estará disponible también como `posts`. Aparte de este pequeño elemento, el código debe sernos familiar:

~~~js
Router.map(function() {
  this.route('postsList', {
    path: '/:postsLimit?',
    waitOn: function() {
      var limit = parseInt(this.params.postsLimit) || 5; 
      return Meteor.subscribe('posts', {sort: {submitted: -1}, limit: limit});
    },
    data: function() {
      var limit = parseInt(this.params.postsLimit) || 5; 
      return {
        posts: Posts.find({}, {sort: {submitted: -1}, limit: limit})
      };
    }
  });
  
  //..
});
~~~
<%= caption "lib/router.js" %>
<%= highlight "8~13" %>

Ahora que hemos establecido el contexto de datos a nivel de router podemos deshacernos del ayudante de plantilla `posts` del archivo `posts_list.js`. Y como hemos llamado `posts` al contexto de datos (igual que en el ayudante), ¡ni siquiera necesitamos tocar la plantilla `postsList`!

Recapitulemos. Así es como ha quedado nuestro `router.js`:

~~~js
Router.configure({
  layoutTemplate: 'layout',
  loadingTemplate: 'loading',
  waitOn: function() { 
    return [Meteor.subscribe('notifications')]
  }
});

Router.map(function() {
  //...

  this.route('postsList', {
    path: '/:postsLimit?',
    waitOn: function() {
      var limit = parseInt(this.params.postsLimit) || 5; 
      return Meteor.subscribe('posts', {sort: {submitted: -1}, limit: limit});
    },
    data: function() {
      var limit = parseInt(this.params.postsLimit) || 5; 
      return {
        posts: Posts.find({}, {sort: {submitted: -1}, limit: limit})
      };
    }
  });
});
~~~
<%= caption "lib/router.js" %>
<%= highlight "5, 11~21" %>

<%= commit "12-2", "Aumentada la ruta `postsList` para que tenga un límite." %>

Vamos a probar nuestro nuevo sistema de paginación. Ahora podemos mostrar un número arbitrario de posts en la página principal simplemente cambiando el parámetro en la URL. Por ejemplo, intenta acceder a `http://localhost:3000/3`. Deberías ver algo como esto:

<%= screenshot "12-2", "Controlling the number of posts on the homepage. " %>

<% note do %>

### ¿Y porqué no usamos páginas?

¿Por qué usamos el enfoque "paginación infinita" en lugar de mostrar páginas sucesivas con 10 posts cada uno, como hace Google para en sus resultados de búsqueda? En realidad se debe al paradigma de tiempo real que utiliza Meteor.

Imaginemos que paginamos nuestra colección `Posts` utilizando el patrón de resultados Google, y que estamos en la página 2, que muestra los mensajes de 10 a 20. ¿Qué pasa si otro usuario elimina una de las 10 entradas anteriores?

Como nuestra aplicación es en tiempo real, nuestra base de datos cambiaría. El post 10 se convertiría en el 9, y desaparecería de nuestra vista y ahora veríamos el 11. El resultado sería que el usuario vería aparecer y desaparecer posts sin razón aparente.

Incluso si toleramos esta peculiaridad, la paginación tradicional también es difícil de implementar por razones técnicas.

Volvamos a nuestro ejemplo anterior. Estamos publicando los posts 10-20 de la colección `Posts`, pero ¿cómo los encontramos en el cliente? No se pueden seleccionar los mensajes 10 a 20 porque sólo hay diez posts en total en el conjunto de datos del lado del cliente.

Una solución sería publicar esos 10 mensajes en el servidor y, a continuación, hacer un `Posts.find()` en el lado del cliente para recoger todos los posts publicados.

Esto funciona si sólo tienes una suscripción. Pero, ¿y si tenemos más de una?

Digamos que una suscripción pide los posts 10 a 20, y otra 30 a 40 . Ahora tenemos 20 mensajes cargados del lado del cliente en total, y no hay forma de saber cuáles pertenecen a cada suscripción.

Por todas estas razones, la paginación tradicional simplemente no tiene mucho sentido cuando se trabaja con Meteor.

<% end %>

### Creando un controlador de rutas

Te habrás dado cuenta de que repetimos dos veces la línea `var limit = parseInt(this.params.postsLimit) || 5;`. Además, codificar el número "5" no es lo ideal. No te preocupes, no es el fin del mundo, pero como siempre es mejor seguir el principio DRY (No Repeat Yourself), vamos a ver cómo podemos refactorizar un poco las cosas.

Introduciremos un nuevo aspecto de Iron Router, los _controladores de ruta_. Un controlador de ruta es simplemente una forma de agrupar en un paquete reutilizable, características de enrutamiento que puede heredar cualquier ruta. Ahora sólo lo utilizaremos para una sola ruta, pero en el próximo capítulo veremos que esta característica es muy útil.

~~~js
PostsListController = RouteController.extend({
  template: 'postsList',
  increment: 5, 
  limit: function() { 
    return parseInt(this.params.postsLimit) || this.increment; 
  },
  findOptions: function() {
    return {sort: {submitted: -1}, limit: this.limit()};
  },
  waitOn: function() {
    return Meteor.subscribe('posts', this.findOptions());
  },
  data: function() {
    return {posts: Posts.find({}, this.findOptions())};
  }
});

Router.map(function() {
  //...

  this.route('postsList', {
    path: '/:postsLimit?',
    controller: PostsListController
  });
});
~~~
<%= caption "lib/router.js" %>

Vamos a verlo paso a paso. En primer lugar, estamos creamos nuestro controlador extendiendo `RouteController`. A continuación, establecemos la propiedad `template` tal y como hicimos antes, y luego una nueva propiedad  `increment`.

A continuación, definimos una nueva función de `limit` que devolverá el límite actual, y una función `findOptions` que devolverá un objeto con las opciones. Esto puede parecer un paso extra, pero vamos a hacer uso de él en el futuro.

A continuación, definimos nuestras funciones `waitOn` y de `data` igual que antes, excepto que ahora usan la nueva función `findOptions`.

Lo último es decirle a la ruta `postsList` que use el nuevo controlador con la propiedad `controller`.

<%= commit "12-3", "postLists refactorizado en un controlador de rutas" %>

### Añadiendo un botón "Load more"

Ya tenemos funcionando la paginación. Sólo hay un problema: no hay manera de utilizarla realmente si no escribimos en la URL manualmente. Desde luego, no parece una gran experiencia de usuario, así que vamos a arreglarlo.

Lo que queremos hacer es bastante simple. Vamos a añadir un botón "load more" en la parte inferior de nuestra lista de posts, que se incrementará en 5 el número de posts que se muestran. Así que si estamos en la URL `http://localhost:3000/5`, haciendo clic en "load more" debería llevarnos a `http://localhost:3000/10`. Si has llegado hasta aquí, confiamos en que podrás manejar un poco de aritmética!

Al igual que antes, vamos a añadir nuestra lógica de paginación en la ruta. ¿Recuerdas que nombramos explícitamente el contexto de datos en lugar de usar un cursor anónimo? Bueno, pues no hay ninguna regla que diga que la función `data` sólo puede pasar cursores, de modo que usaremos la misma técnica para generar la URL del botón "load more".

~~~js
PostsListController = RouteController.extend({
  template: 'postsList',
  increment: 5, 
  limit: function() { 
    return parseInt(this.params.postsLimit) || this.increment; 
  },
  findOptions: function() {
    return {sort: {submitted: -1}, limit: this.limit()};
  },
  waitOn: function() {
    return Meteor.subscribe('posts', this.findOptions());
  },
  posts: function() {
    return Posts.find({}, this.findOptions());
  },
  data: function() {
    var hasMore = this.posts().fetch().length === this.limit();
    var nextPath = this.route.path({postsLimit: this.limit() + this.increment});
    return {
      posts: this.posts(),
      nextPath: hasMore ? nextPath : null
    };
  }
});
~~~
<%= caption "lib/router.js" %>
<%= highlight "16~23" %>

Echemos un vistazo en profundidad a este pequeño truco de magia que hemos puesto en el router. Recuerda que la ruta `postsList` (que se hereda del controlador `PostsListController` con el que estamos trabajando) toma un parámetro `postsLimit`.

Cuando alimentamos `{postsLimit: this.limit() + this.increment}` a `this.route.path()`, le estamos diciendo a la ruta `postsList` que  construya su propio path utilizando ese objeto JavaScript como contexto de datos.

En otras palabras, es exactamente lo mismo que usar el ayudante Handlebars `{{'postsList' pathFor}}`, salvo que reemplazamos el `this` implícito por nuestro propio contexto de datos a medida.

Estamos cogiendo ese path y añadiendolo al contexto de datos de nuestra plantilla, pero _sólo_ si hay más posts que mostrar. La forma de hacerlo es un poco complicada.

Sabemos que `this.limit()` devuelve el número actual de posts que nos gustaría mostrar, que puede ser el valor de la URL actual, o el valor por defecto (5) si la URL no contiene ningún parámetro.

Por otro lado, `this.posts` se refiere al cursor actual, de modo que `this.posts.fetch().length` es el número de mensajes que hay en el cursor.

Así que lo que estamos diciendo es que si pedimos `n` posts y obtenemos `n`, seguiremos mostrando el botón "load more". Pero si pedimos `n` y tenemos _menos_ de `n`, significa que hemos llegado al límite y deberíamos dejar de mostrarlo.

Con todo esto, nuestro sistema falla en un caso: cuando el número de posts en nuestra base de datos es _exactamente_ `n`. Si eso ocurre, el cliente pedirá `n` posts y obtendrá `n` por lo que seguirá mostrando el botón "load mores", sin darse cuenta de que ya no quedan más elementos.

Lamentablemente, no hay soluciones sencillas para este problema, así que por ahora vamos a tener que conformarnos con esto.

Todo lo que queda por hacer es añadir el botón "load more" en la parte inferior de nuestra lista de posts, asegurándonos de mostrarlo sólo si tenemos más posts que cargar:

~~~html
<template name="postsList">
  <div class="posts">
    {{#each posts}}
      {{> postItem}}
    {{/each}}
    
    {{#if nextPath}}
      <a class="load-more" href="{{nextPath}}">Load more</a>
    {{/if}}
  </div>
</template>
~~~
<%= caption "client/views/posts/posts_list.html" %>
<%= highlight "7~10" %>

Así es como se debería ver la lista ahora:

<%= screenshot "12-3", "El botón “load more”." %>

<%= commit "12-4", "Añadido nextPath() al controlador para desplazarnos por la lista de posts" %>


<% note do %>

### Count vs Length

Podrías preguntarte por qué usamos `this.posts.fetch().length` en lugar de `this.posts.count()`. Esto es sólo una solución temporal para hacer frente a un bug en Meteor presente en el momento de escribir esto. ¡Esperemos que se arregle pronto!

<% end %>

### Mejorando la barra de progreso

La paginación funciona correctamente, pero tiene una peculiaridad algo molesta: cada vez que se hace clic en "load more" y el router pide más posts, nos envía a plantilla de la carga mientras esperamos los nuevos datos. El resultado es que cada vez, nos envía a la parte superior de la página y tenemos que desplazarnos hasta el final para reanudar la navegación.

Sería mucho mejor si pudiéramos quedarnos en la misma página durante toda la operación, sin dejar de ofrecer algún tipo de indicación de que se están cargando nuevos datos. Afortunadamente, eso es precisamente lo que hace el paquete `iron-router-progress`.

Al igual que en el Safari de iOS o sitios como Medium y YouTube, el paquete `iron-router-progress` agrega una barra de carga delgada en la parte superior de la pantalla. La implementación es tan fácil como añadir el paquete a la aplicación:

~~~bash
mrt add iron-router-progress
~~~
<%= caption "Terminal" %>

Gracias a la magia de los paquetes inteligentes, ¡el nuevo indicador de progreso funcionará nada más instalarlo! La barra de progreso se activará para cada ruta, y se completará automáticamente una vez que los datos requeridos de la ruta se hayan cargado.

Sólo utilizaremos un pequeño truco. Vamos a desactivar `iron-router-progress` para la ruta `postSubmit` ya que esta ruta no tiene que esperar datos (después de todo, es sólo un formulario vacío):

~~~js
Router.map(function() {
 
  //...
  
  this.route('postSubmit', {
    path: '/submit',
    disableProgress: true
  });
});
~~~
<%= caption "lib/router.js" %>
<%= highlight "7" %>

<%= commit "12-5", "Usando el paquete `iron-router-progress` para mejorar la paginacion." %>

*Nota del traductor:*

*Podríamos deshabilitar también la barra de progreso para la ruta `postEdit`.*

    this.route('postEdit', {
      path: '/posts/:_id/edit',
      disableProgress: true
    });

*Además, hay que resaltar que se nota un aumento considerable en el consumo de CPU del navegador al hacer uso del paquete `iron-router-progress`. Hay una [issue abierta al respecto en en el repositorio del paquete](https://github.com/Multiply/iron-router-progress/issues/14).*

### Accediendo a cualquier post

Por defecto, cargamos los cinco últimos posts, pero ¿qué pasa si vamos a la página de un post individual?

<%= screenshot "12-4", "Una plantilla vacía." %>

Si lo pruebas, verás que sólo hay una plantilla de vacía. en realidad, tiene sentido: le hemos dicho al router que se suscriba a la publicación `posts` cuando carga la ruta `postsList`, pero no le hemos dicho qué debe hacer con la ruta `postpage`.

Pero hasta el momento, lo único que sabemos es suscribirnos a una lista de los `n` últimos posts. ¿Cómo nos pedimos al servidor un solo post? Te contaré un pequeño secreto: ¡podemos usar más de una publicación para cada colección!

Así que para volver a ver los posts perdidos, haremos una nueva publicación `singlePost` que sólo publica un post, identificado por `_id`. 

~~~js
Meteor.publish('posts', function(options) {
  return Posts.find({}, options);
});

Meteor.publish('singlePost', function(id) {
  return id && Posts.find(id);
});
~~~
<%= caption "server/publications.js" %>
<%= highlight "5~7" %>

Ahora, vamos a suscribirnos a los posts correctos en el lado del cliente. Ya estamos suscritos a la publicación `comments` en la función `waitOn` de la ruta `postPage`, por lo que simplemente podemos añadir ahí la suscripción a `singlePost`. Sin olvidarnos de añadir la suscripción a la ruta `postEdit`, que también necesita los mismos datos:

~~~js
Router.map(function() {

  //...
    
  this.route('postPage', {
    path: '/posts/:_id',
    waitOn: function() {
      return [
        Meteor.subscribe('singlePost', this.params._id),
        Meteor.subscribe('comments', this.params._id)
      ];
    },
    data: function() { return Posts.findOne(this.params._id); }
  });

  this.route('postEdit', {
    path: '/posts/:_id/edit',
    waitOn: function() { 
      return Meteor.subscribe('singlePost', this.params._id);
    },
    data: function() { return Posts.findOne(this.params._id); }    
  });
  
  /...

});
~~~
<%= caption "lib/router.js" %>
<%= highlight "7~12,18~20" %>

<%= commit "12-6"," Usando una sola suscripción a los posts para asegurarnos de que siempre se puede ver y editar el post correcto." %>

Terminada la paginación, nuestra aplicación ya no sufre de problemas de escala, y los usuarios pueden contribuir con muchos más posts. ¿No estaría bien tener una forma de clasificarlos? Este es precisamente el tema del siguiente capítulo, los *votos*.