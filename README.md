# IBMCloudFunctions
Tutorial introductorio a IBM Cloud Functions. En este ejemplo, vamos a crear un asistente que informe sobre el estado del clima, utilizando una API del Servicio Meteorológico Nacional de la República Argentina (SMN), Watson Assistant e IBM Cloud™ Functions.


## Prerequisitos
Tener una cuenta en IBM Cloud. En caso de no tenerla aún, podés registrarte en https://cloud.ibm.com/registration

## ¿Qué es IBM Cloud™ Functions?
IBM Cloud™ Functions es la plataforma de functions-as-a-service (FaaS) de IBM Cloud™. Te permite crear código en una diversidad de lenguajes de programación que ejecute la lógica de tu aplicación de manera escalable, sin tener que preocuparte ni ocuparte por la infraestructura. La plataforma de programación de función como servicio (Faas) se basa en el proyecto de código abierto Apache OpenWhisk.

## Parte 1: IBM Cloud™ Functions
1. Ingresá a tu cuenta de [IBM Cloud](https://cloud.ibm.com/login)
1. En el catálogo, buscá _Functions_ y creá una instancia.
![Functions in Catalog](/img/1FunctionsCatalog.png)
1. Hacé click en el botón _Start Creating_
![Start creating](/img/2FunctionsPage.png)
1. Elegí la opción _Create Action_. Indicá un nombre para la acción y el runtime que vas a usar. En mi caso, voy a usar Node.js. Hacé click en _Create_
![Create action](/img/3CreateAction.gif)
1. Te va a aparecer el editor de texto.
![Code editor](/img/4ActionDefault.png) <br> Reemplazá el código que viene por default por el código que aparece abajo y guardá los cambios con el botón _Save_. <br> 
```node.js
var request = require("request");
 
function main(params) {
   var options = {
      url: "https://ws.smn.gob.ar/map_items/weather",//API del servicio meteorológico nacional
      json: true
   };
    //Llamar a la API 
   return new Promise(function (resolve, reject) {
      request(options, function (err, resp) { //La llamada a la API devuelve un JSON que contiene información de todas las ciudades. 
         if (err) {
            console.log(err);
            return reject({err: err});
         }
    
    
    // Voy a buscar en el array el elemento cuyo ['name'] sea igual la ciudad que venga como parametro (que también es un JSON) del assistant
    for(var i = 0; i < resp.body.length; i++)
    {
      if(resp.body[i].name == params.city)
      {
        return resolve({clima: resp.body[i]['weather'].description, temperatura: resp.body[i]['weather'].temp});
      }
    }
      return resolve({clima: 'No pude encontrar información para la ciudad en cuestión'});
      });
   });
}
``` 
  El código usa el package request de Node.js para conectarse con la API del SMN. Esa API devuelve un JSON array con información climática de múltiples ciudades de la República Argentina.
  Usamos Promise para invocar a la API
  Se busca en el array el elemento cuyo atributo 'name' sea igual la ciudad que vamos a pasar como parámetro desde el assistant
  De encontrarse, la función devuelve un JSON con la descripción del clima y la temperatura.

6. En el menú de la izquierda, elegí la opción _Endpoints_ y marcá el checkbox que dice _Enable as Web Action_ Esto va a crear una URL que te va a permitir llamar a tu función desde cualquier aplicación web; y en nuestro caso, desde el assistant.
Copiá la URL que se genera y guardala para más tarde.
![Endpoints](/img/5Endpoints.gif)

## Parte 2: Watson Assistant
1. En el catálogo, buscá _Assistant_ y creá una instancia. Seleccioná el plan _Lite_, poné el nombre que más te guste para el servicio y dale click a _Create_
![Assistant Catalog](/img/6Assistant.png)
1. Ingresá a la interfaz de Watson Assistant haciendo click en el botón _Launch Watson Assistant_
![Launch Watson Assistant](/img/7AssistantPage.png)
1. Ingresá al asistente que se crea por defecto (_My first assistant_)  y hacé click en el icono de _Skills_  (segundo del menú que está a la izquierda)
![Skills](/img/9MyFirstAssistant.png)
![Create skill](/img/10Skills.png)

A partir de acá tenemos dos opciones:
* Podés importar este skill y tener tu asistente armado en un solo paso 
o 
* Armar tu asistente manualmente con estas instrucciones

### Importar skill
Importá el archivo [skill-Clima.json](/skill-Clima.json)
![Importar skill](/img/11ImportSkill.gif)

### Armar asistente de manera manual
1. Creá un nuevo skill en español.
![Crear skill](/img/12CreateSkill.gif)
1. Una vez creado, vas a ver la pantalla de _Intents_ (intenciones) 
![Intents](/img/13Intents.png)
<br>Las Intenciones son las acciones que el usuario del asistente desea realizar, en nuestro caso, obtener información del clima. Al reconocer la intención expresada en una entrada de usuario, el servicio Watson Assistant puede elegir el flujo de diálogo correcto para responder a la misma.<br>Crea una nueva intención llamada `Pedir_Datos_Clima`.

![Intents](/img/14IntentName.png)

Cargale ejemplos de cómo un usuario pediría esa información.
![Intents](/img/15IntentExamples.gif)
1. Ingresá a la opción _Entities_ (entidades) <br> Las entidades representan a los sustantivos: el objeto o el contexto de una acción. En nuestro caso, las ciudades que sabemos que la API del SMN reporta. <br> Por eso, vamos a crear una entidad llamada `Ciudad` y le vamos a cargar la lista de ciudades. Como en total son más de 200, te recomiendo que cargues solo las que vas a usar por el momento, o que vuelvas a la opción de Importar el skill donde ya están todas cargadas.
![Entities](/img/16Entities.png)
1. Ingresá a la opción _Dialog_ (Diálogo) Por defecto, ya vienen los nodos "Bienvenido" y "en otras cosas".
![Default dialog](/img/17DialogStart.png)
<br>Vamos a crear un nuevo nodo al que voy a llamar `Pedir_Info_Clima`. 
    1. Como condición del nodo, agregá `#Pedir_Datos_Clima` (o el nombre que hayas usado para tu intent)<br>
    ![Create node](/img/18CreateNode.gif) 
    <br>
    1. Desde la opción _Customize_ habilitá los slots para ese nodo. Esto permite que el assistant pueda contestar con los datos de la ciudad ingresada o pedir que se ingrese una ciudad si el usuario no lo hizo.<br>
    1. Esto habilita una nueva sección en donde vas a tener que configurar que el assistant chequee por `@ciudad`. Si el usuario indicó una ciudad, la misma se guarda en la variable de entorno `$ciudad`. Si no, el assistant va a preguntar por ella con el texto que definamos en _If not present, then ask_
    ![Create node](/img/19SlotConfig.gif)
    <br>
1. Ahora vamos al menú _Options > Webhooks_. En URL vas a tener que agregar la URL pública de tu función, que guardaste en el punto 6 de la Parte 1. Al final de la URL agregá `.json`
![Create node](/img/20Webhooks.png)
1. Volvé a tu nodo `Pedir_Info_Clima` y creá un child node que se llame `API_call` y en _Customize_, habilitá la opción "Webhook". Esto va a habilitar otra sección nueva en el nodo para incluir la información que le vamos a mandar a la función como parámetro.
*Key* `city` *Value* `El clima en $ciudad está $webhook_result_1.clima`
![Create node](/img/21Webhookconfig.gif)
Una vez que están configurados los parámetros, en la respuesta, incluí el texto que quieras responder. Por ejemplo:
*If assistant recognizes* `$webhook_result_1` *Respond with* `El clima en $ciudad está $webhook_result_1.clima`
1. Volvé al nodo `Pedir_Info_Clima` y asegurate de setear para que saltee la interacción con el usuario.
![Create node](/img/22JumpNode.png)

### Bonus Track:
La función que armamos también devuelve en el JSON la temperatura. Podés chequear en el nombre.skill para saber como crear respuestas condicionales en base al valor de esa temperatura.

![Create node](/img/TryItOut.gif)

## Material de Referencia:
Terminología IBM Cloud™ Functions: https://cloud.ibm.com/docs/openwhisk?topic=cloud-functions-about&locale=es
How to invoke an external REST API from a cloud function: https://maxkatz.org/2018/07/20/how-to-invoke-an-external-rest-api-from-a-cloud-function/
Iniciación a Functions: https://cloud.ibm.com/docs/openwhisk?topic=cloud-functions-getting-started&locale=es
Información sobre la API del SMN: http://foro.gustfront.com.ar/viewtopic.php?t=5252
Iniciación a Watson Assistant: https://cloud.ibm.com/docs/services/assistant?topic=assistant-getting-started&locale=es
Watson Assistant Webhooks: https://cloud.ibm.com/docs/services/assistant?topic=assistant-dialog-webhooks

## Troubleshooting:
- *Problema* Cuando llamo a la API me sale _The callout generated this error: {"message":"Connection to Webhook failed. No response."}._
*Solución* Andá a la parte de webhooks de Watson Assistant y agregá `.json` al final de la URL de tu función.

