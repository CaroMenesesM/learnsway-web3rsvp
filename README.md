# README en español
# **Crear una aplicación de Confirmación de asistencia (RSVP) a través de Sway en Fuel**

En este taller vamos a crear una dapp completa (fullstack) en Fuel.

Esta dapp es un referente arquitectónico básico para una plataforma de gestíon y creación de eventos, similar a Eventbrite o Lu.ma. Los usuarios podrán crear un nuevo evento y confirmar su asistencia (*hacer RSVP*) a un evento existente. Estas son las funcionalidades que vamos a crear en este taller:

* Crear una función en el contrato inteligente para crear un nuevo evento;
* Crear una función en el contrato inteligente para confirmar asistencia (*hacer RSVP*) a un evento ya existente.
 (#PRIMERA IMAGEN AQUI)
 
 Vamos a descomponer las tareas individuales asociadas a cada función:
 
 *Para poder crear una función que de origen a un nuevo evento, el programa debe ser capaz de manejar lo siguiente:*
*  En nombre del evento, el usuario debe poder pasar una suma como depósito que los asistentes paguen para poder confirmar su asistencia al evento, y la capacidad máxima del evento.
*  Una vez el usuario transfiere esta información, el programa debe crear un evento representado en una estructura de datos llamada `struct`.
*  Como esta es una plataforma de eventos, nuestro programa debe ser capaz de manejar múltiples eventos a la vez. Por lo tanto, necesitamos un mecanismo que almacene múltiples eventos.
*  Para almacenar múltiples eventos, vamos a usar un mapa hash, también conocido como tabla hash en otros lenguajes de programación. Este mapa hash asignará/"mapeará" `map` un identificador único que llamaremos `eventId` a un evento (que es representado como un struct).

*Para crear una función que maneje la confirmación de asistencia a un evento de un usuario determinado, nuestro programa deberá ser capaz de manejar lo siguiente:*
* Debemos tener un mecanismo que identifique el evento cuya asistencia el usuario quiere confirmar.

Algunos recursos que pueden ser útiles:
*  La guía de fuel
*  La guía de Sway
*  El Discord de fuel - para obtener ayuda.

# **Instalación**

1. Instale `cargo` utilizando [rustup](https://www.rust-lang.org/tools/install)

Mac y Linux:
`curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh`

2. Compruebe que la configuración esté correcta:
```
$ cargo --version
cargo 1.62.0
```

3. Instale `forc` utilizando [fuelup](https://fuellabs.github.io/sway/v0.18.1/introduction/installation.html#installing-from-pre-compiled-binaries)

Mac y Linux:
```
curl --proto '=https' --tlsv1.2 -sSf \
https://fuellabs.github.io/fuelup/fuelup-init.sh | sh
```

5. Compruebe que la configuración esté correcta:
```
$ forc --version
forc 0.26.0
```

## **Editor de código**
Puede utilizar el editor que prefiera.
* [Plugin/complemento para VSCode](https://marketplace.visualstudio.com/items?itemName=FuelLabs.sway-vscode-plugin)
* [Resaltado de vim](https://github.com/FuelLabs/sway.vim)


# **Pasos iniciales**
Esta guía orientará a los desarrolladores para poder redactar un contrato inteligente en el lenguaje Sway, realizar una prueba sencilla, desplegar el contrato en Fuel y crear una interfaz/frontend.

Antes de empezar, puede ser útil comprender la terminología que será usada a lo largo de la documentación y cómo se relaciona entre si:

* **Fuel**: la cadena de bloques de Fuel.
* **FuelVM**: la máquina virtual que impulsa a Fuel.
* **Sway**: el lenguaje específico de dominio diseñado para FuelVM; está inspirado en Rust.
* **Forc**: el sistema de compilación y administrador de paquetes para Sway, similar a Cargo en el caso de Rust.

## **Entendiendo los tipos de programas de Sway**
Hay cuatro tipos de programas de Sway:

* `contract` (*contrato*)
* `predicate` (*predicado*)
* `script` (*script*)
* `library` (*biblioteca*)

Los contratos, los predicados y los scripts pueden producir objetos utilizables en la cadena de bloques, mientras que una biblioteca es simplemente un proyecto diseñado para la reutilización de código y no se puede desplegar directamente.

Las principales características de un contrato inteligente que lo diferencian de los scripts o predicados son: que aquel es invocable y posee un estado activamente.

Un script es un código de *bytes* ejecutable en la cadena que puede invocar contratos específicos para realizar alguna tarea. No tiene la propiedad de ningún recurso y no puede ser invocado por un contrato.

# **Crear un nuevo proyecto en Fuel**

Comience creando una nueva carpeta vacía. La llamaremos `Web3RSVP`

## **Creación del contrato**

Luego, con `forc` instalado, cree un proyecto de contrato dentro de su carpeta `Web3RSVP`:
```
$ cd Web3RSVP
$ forc new eventPlatform
To compile, use `forc build`, and to run tests use `forc test`

---
```
Aquí está el proyecto inicializado por `Forc`:

```
$ tree Web3RSVP
eventPlatform
├── Cargo.toml
├── Forc.toml
├── src
│   └── main.sw
└── tests
    └── harness.rs
```

# **Definiendo la IBA Interfaz binaria de aplicaciones (ABI, por sus siglas en inglés)**

Primero, definiremos la IBA. La IBA define una interfaz y la función no tiene contenido en la IBA. El contrato debe definir o importar una declaración IBA e implementarla. Se considera una buena práctica definir su IBA en una biblioteca separada e importarla a su contrato porque esto permite a los invocadores del contrato importar y usar la IBA en scripts para invocar su contrato.

Para definir la IBA como biblioteca, crearemos un nuevo archivo en la carpeta `src`. Cree un nuevo archivo llamado `event_platform.sw`

Así es como se debe ver su estructura del proyecto hasta ahora: Aquí está el proyecto inicializado por `Forc`:

```
eventPlatform
├── Cargo.toml
├── Forc.toml
├── src
│   └── main.sw
│   └── event_platform.sw
└── tests
    └── harness.rs
```

Agregue el siguiente código a su archivo de IBA, `event_platform.sw`:

```
library event_platform;

use std::{
    identity::Identity,
    contract_id::ContractId,
};

abi eventPlatform {
    #[storage(read, write)]
    fn create_event(max_capacity: u64, deposit: u64, event_name: str[10]) -> Event;

    #[storage(read, write)]
    fn rsvp(event_id: u64) -> Event;
}

// defining the struct here because it would be used by other developers who would be importing this ABI
pub struct Event {
    unique_id: u64,
    max_capacity: u64,
    deposit: u64,
    owner: Identity,
    name: str[10],
    num_of_rsvps: u64,
}

```
Ahora, en el archivo `main.sw` implementaremos estas funciones. Aquí es en donde vamos a escribir el cuerpo de las funciones. Así es como se debe ver su archivo `main.sw` hasta ahora:

```
contract;

dep event_platform;
use event_platform::*;

use std::{
   chain::auth::{AuthError, msg_sender},
    constants::BASE_ASSET_ID,
    context::{
   call_frames::msg_asset_id,
        msg_amount,
        this_balance,
    },
    contract_id::ContractId,
    identity::Identity,
    logging::log,
    result::Result,
    storage::StorageMap,
    token::transfer,
};

storage {
    events: StorageMap<u64, Event> = StorageMap {},
    event_id_counter: u64 = 0,
}

impl eventPlatform for Contract {
    #[storage(read, write)]
    fn create_event(capacity: u64, price: u64, event_name: str[10]) -> Event {
        let campaign_id = storage.event_id_counter;
        let new_event = Event {
            unique_id: campaign_id,
            max_capacity: capacity,
            deposit: price,
            owner: msg_sender().unwrap(),
            name: event_name,
            num_of_rsvps: 0,
        };

        storage.events.insert(campaign_id, new_event);
        storage.event_id_counter += 1;
        let mut selectedEvent = storage.events.get(storage.event_id_counter - 1);
        return selectedEvent;
    }

    #[storage(read, write)]
    fn rsvp(event_id: u64) -> Event {
        let sender = msg_sender().unwrap();
        let asset_id = msg_asset_id();
        let amount = msg_amount();

     // get the event
     //variables are immutable by default, so you need to use the mut keyword
        let mut selected_event = storage.events.get(event_id);

    // check to see if the eventId is greater than storage.event_id_counter, if
    // it is, revert
        require(selected_event.unique_id < storage.event_id_counter, InvalidRSVPError::InvalidEventID);

    // check to see if the asset_id and amounts are correct, etc, if they aren't revert
        require(asset_id == BASE_ASSET_ID, InvalidRSVPError::IncorrectAssetId);
        require(amount >= selected_event.deposit, InvalidRSVPError::NotEnoughTokens);

          //send the payout from the msg_sender to the owner of the selected event
        transfer(amount, asset_id, selected_event.owner);

    // edit the event
        selected_event.num_of_rsvps += 1;
        storage.events.insert(event_id, selected_event);

    // return the event
        return selected_event;
    }
}
```

## **Construir el contrato**

Desde el interior del directorio `web3rsvp/eventPlatform` ejecute el siguiente comando para construir su contrato:

```
$ forc build
  Compiled library "core".
  Compiled library "std".
  Compiled contract "counter_contract".
  Bytecode size is 224 bytes.
```
## **Desplegar el contrato**
Ahora es momento de desplegar el contrato en la red de pruebas (*testnet*). Le enseñaremos cómo hacerlo usando `forc` desde la línea de comandos, pero también puede hacerlo usando el [SDK de Rust](https://github.com/FuelLabs/fuels-rs#deploying-a-sway-contract) o el [SDK de TypeScript](https://github.com/FuelLabs/fuels-ts/#deploying-contracts).

Para poder desplegar un contrato, necesita tener una billetera para firmar la transacción y saldo para pagar el gas. Primero, crearemos una billetera.

### **Instale la interfaz de línea de comando (CLI, por sus siglas en inglés) de la billetera**
Siga [estos pasos para configurar una billetera y crear una cuenta](https://github.com/FuelLabs/forc-wallet#forc-wallet).

Después de escribir una contraseña, asegúrese de guardar la frase mnemónica que se produce.

Con esto, obtendrá una dirección de Fuel que se ve así: `fuel1efz7lf36w9da9jekqzyuzqsfrqrlzwtt3j3clvemm6eru8fe9nvqj5kar8.` Guarde esta dirección, ya que la necesitará para firmar las transacciones cuando despleguemos el contrato.

### **Obtenga fondos en la *testnet***
Con la dirección de su cuenta/billetera, diríjase al [grifo de la *testnet*](https://faucet-beta-1.fuel.network/) para recibir fondos en su billetera.

### **Desplegando a la *Testnet***
Ahora que tiene una billetera, puede desplegar con `forc deploy` y pasar por el *endpoint* de la *testnet*, de la siguiente manera:

`forc deploy --url https://node-beta-1.fuel.network/graphql --gas-price 1`

**Nota:** Debemos establecer el precio del gas en 1. Sin esto, el precio por defecto del gas es 0 y la transacción fallará.

La terminal solicitará la dirección de la billetera con la que usted quiere firmar esta transacción. Copie la dirección que guardó en pasos anteriores al crear la billetera, la que se ve así:

`fuel1efz7lf36w9da9jekqzyuzqsfrqrlzwtt3j3clvemm6eru8fe9nvqj5kar8`

La terminal mostrará el `Contract id`. Asegúrese de guardar esto, ya que lo necesitará para crear la interfaz/el *frontend* con la SDK de Typescript.

La terminal mostrará un identificador de transacción para firmar, `transaction id to sign` y le pedirá la firma. Abra una nueva pestaña en la terminal y revise sus cuentas ejecutando el comando `forc wallet list`. Si ha seguido estos pasos, verá que sólo tiene una cuenta, `0`.

Tome el `transaction id` desde la otra pestaña de la terminal y firme con su cuenta ejecutando el siguiente comando:

`forc wallet sign` + `[transaction id here, without brackets]` + `[the account number, without brackets]`

Su comando deberá verse así:

```
$ forc wallet sign 16d7a8f9d15cfba1bd000d3f99cd4077dfa1fce2a6de83887afc3f739d6c84df 0
Please enter your password to decrypt initialized wallet's phrases:
Signature: 736dec3e92711da9f52bed7ad4e51e3ec1c9390f4b05caf10743229295ffd5c1c08a4ca477afa85909173af3feeda7c607af5109ef6eb72b6b40b3484db2332c
```

Ingrese su contraseña cuando se la soliciten y obtendrá una `signature`. Guarde dicha firma, regrese a su otra ventana de la terminal, y péguela en donde se la solicitan: `provide a signature for this transaction`

Finalmente, recibirá una `TransactionId` que confirma que su contrato se desplegó exitosamente. Con este ID, puede dirigirse al [explorador de bloques](https://fuellabs.github.io/block-explorer-v2/) y ver su contrato.

**Nota:** Para poder ver la transacción en el explorador de bloques, debe agregar `0x` como prefijo de su `TransactionID`


## **Crear una interfaz/un frontend que interactúe con el Contrato**
Ahora, vamos a:
**1. Inicializar un projecto React.
2. Instalar las dependencias del SDK de `fuels`.
3. Modificar la App.
4. Ejecutar nuestro proyecto.**

### **Inicializar un projecto React**
Para dividir mejor nuestro proyecto, creemos una nueva carpeta llamada `frontend` e inicialicemos nuestro proyecto desde la misma.

En la terminal, devuélvase un nivel en el directorio e inicialice un proyecto React usando la [App *Create React*](https://create-react-app.dev/)
```
$ cd ..
$ npx create-react-app frontend --template typescript
Success! Created frontend at Fuel/WEB3RSVP/frontend
```
Ahora debe tener la carpeta externa, `Web3RSVP`, con dos carpetas en su interior: `frontend` Y `rsvpContract`

# **Translator's note: Cami, for some reason this image appears to have a broken link**

### Instale las dependencias del SDK fuels
En este paso, necesitamos instalar 3 dependencias para el proyecto:

1. `fuels`: El paquete sombrilla que incluye todas las herramientas principales: `Wallet`, `Contracts`, `Providers` y otros.
2. `fuelchain`: Fuelchain es el generador de IBA en TypeScript.
3. `typechain-target-fuels`: El *Fuelchain Target* es el interpretador específico para la especificación de la IBA.

IBA significa Interfaz Binaria de Aplicaciones. Las IBAs le facilitan a la aplicación la interfaz para interactuar con la Máquina Virtual; en otras palabras, proporcionan información a la App tal como qué métodos tiene un contrato, qué parámetros, tipos esperados, etc...

## **Instalación**

Ingrese a la carpeta `frontend`, e instale las siguientes dependencias:

$ cd frontend
$ npm install fuels --save
added 65 packages, and audited 1493 packages in 4s
$ npm install fuelchain typechain-target-fuels --save-dev
added 33 packages, and audited 1526 packages in 2s

## **Generando las clases de contratos**

Para facilitar la interacción con nuestro contrato, utilizamos `fuelchain` para interpretar el resultado de IBA JSON de nuestro contrato. Este JSON fue creado en el momento en que ejecutamos `forc build` para compilar nuestro Contrato de Sway en forma binaria.

Si puede ver la carpeta `Web3RSVP/rsvpContract/out` podrá ver el IBA JSON allí. Si quiere aprender más, lea las [especificaciones de IBA aquí:](https://fuellabs.github.io/fuel-specs/master/protocol/abi/index.html).

Al interior del directorio `Web3RSVP/frontend `ejecute:

```
$ npx fuelchain --target=fuels --out-dir=./src/contracts ../rsvpContract/out/debug/*-abi.json
Successfully generated 4 typings!
```
Ahora deberá poder encontrar una nueva carpeta `Web3RSVP/frontend/src/contracts`. Esta carpeta fue auto-generada por nuestro comando `fuelchain`. Estos archivos abstraen el trabajo que se requiere para crear una instancia de contrato, y generar una interfaz completa en TypeScript para interactuar con el Contrato, facilitando el desarrollo.

### **Crear una billetera (de nuevo)**
Para interactuar con la red de Fuel, debemos enviar transacciones firmadas con suficientes fondos para cubrir los gastos que conlleva usar la red. El SDK de TypeScript de Fuel no soporta integraciones con billeteras actualmente, lo que hace que debamos mantener una billetera no segura al interior de la WebApp utilizando una llave privada.

**Nota**: Esto sólo debe hacerse con propósitos de desarrollo. Nunca exponga una aplicación web con una llave privada al interior. La billetera Fuel está en constante desarrollo, siga su progreso [aquí](https://github.com/FuelLabs/fuels-wallet).

**Nota**: El equipo está trabajando para simplificar el proceso de crear una billetera y eliminar la necesidad de crearla dos vces. Esté atento a estas actualizaciones.

En la raíz del proyecto *Frontend* cree un archivo llamado `createWallet.js`

```
const { Wallet } = require("fuels");

const wallet = Wallet.generate();

console.log("address", wallet.address.toString());
console.log("private key", wallet.privateKey);
```

En la terminal, ejecute el siguiente comando:

```
$ node createWallet.js
address fuel160ek8t7fzz89wzl595yz0rjrgj3xezjp6pujxzt2chn70jrdylus5apcuq
private key 0x719fb4da652f2bd4ad25ce04f4c2e491926605b40e5475a80551be68d57e0fcb
```
**Nota:** Debe usar la dirección y llave privada generados.

Guarde la dirección generada y la llave privada, pues la necesitará luego para establecerla como un valor del string para una variable `WALLET_SECRET` en su archivo `App.tsx`. Más información al respecto a continuación.

Primero, tome la dirección de su billetera y utilícela para recibir fondos del [grifo de la *Testnet*.](https://faucet-beta-1.fuel.network/)

Ahora está listo para crear y publicar ⛽

### **Modificar la App**
Al interior de la carpeta `frontend`, vamos a agregar código que interactúe con nuestro contrato. Lea los comentarios para entender mejor y más fácilmente las partes de la App. 

Cambie el archivo `Web3RSVP/frontend/src/App.tsx` a: 

```
import React, { useEffect, useState } from "react";
import { Wallet } from "fuels";
import "./App.css";
// Import the contract factory -- you can find the name in index.ts.
// You can also do command + space and the compiler will suggest the correct name.
import { RsvpContractAbi__factory } from "./contracts";
// The address of the contract deployed the Fuel testnet
// const CONTRACT_ID = "0x32f10d6f296fbd07e16f24867a11aab9d979ad95f54b223efc0d5532360ef5e4";
const CONTRACT_ID = "<YOUR-CONTRACT-ADDRESS-HERE>";
//the private key from createWallet.js
const WALLET_SECRET = "<YOU-PRIVATE-KEY-HERE>"
// Create a Wallet from given secretKey in this case
// The one we configured at the chainConfig.json
const wallet = new Wallet(WALLET_SECRET, "https://node-beta-1.fuel.network/graphql");
// Connects out Contract instance to the deployed contract
// address using the given wallet.
const contract = RsvpContractAbi__factory.connect(CONTRACT_ID, wallet);

export default function App(){
  const [loading, setLoading] = useState(false);
  //-----------------------------------------------//
  //state variables to capture the selection of an existing event to RSVP to
  const [eventName, setEventName] = useState('');
  const [maxCap, setMaxCap] = useState(0);
  const [deposit, setDeposit] = useState(0);
  const [eventCreation, setEventCreation] = useState(false);
  const [rsvpConfirmed, setRSVPConfirmed] = useState(false);
  const [numOfRSVPs, setNumOfRSVPs] = useState(0);
  const [eventId, setEventId] = useState('');
  //-------------------------------------------------//
  //state variables to capture the creation of an event
  const [newEventName, setNewEventName] = useState('');
  const [newEventMax, setNewEventMax] = useState(0);
  const [newEventDeposit, setNewEventDeposit] = useState(0);
  const [newEventID, setNewEventID] = useState('')
  const [newEventRSVP, setNewEventRSVP] = useState(0);

  useEffect(() => {
    console.log('Wallet address', wallet.address.toString());
    wallet.getBalances().then(balances => {
      const balancesFormatted = balances.map(balance => {
        return [balance.assetId, balance.amount.format()];
      });
      console.log('Wallet balances', balancesFormatted);
    });
  }, []);

  useEffect(() => {
    console.log("eventName", eventName);
    console.log("deposit", deposit);
    console.log("max cap", maxCap);
  },[eventName, maxCap, deposit]);

  async function rsvpToEvent(){
    setLoading(true);
    try {
      console.log('amount deposit', deposit);
      const { value, transactionResponse, transactionResult } = await contract.functions.rsvp(eventId).callParams({
        forward: [deposit]
        //variable outputs is when a transaction creates a new dynamic UTXO
        //for each transaction you do, you'll need another variable output
        //for now, you have to set it manually, but the TS team is working on an issue to set this automatically
      }).txParams({gasPrice: 1, variableOutputs: 1}).call();
      console.log(transactionResult);
      console.log(transactionResponse);
      console.log("RSVP'd to the following event", value);
      console.log("deposit value", value.deposit.toString());
      console.log("# of RSVPs", value.num_of_rsvps.toString());
      setNumOfRSVPs(value.num_of_rsvps.toNumber());
      setEventName(value.name.toString());
      setEventId(value.unique_id.toString());
      setMaxCap(value.max_capacity.toNumber());
      setDeposit(value.deposit.toNumber());
      //value.deposit.format()
      console.log("event name", value.name);
      console.log("event capacity", value.max_capacity.toString());
      console.log("eventID", value.unique_id.toString())
      setRSVPConfirmed(true);
      alert("rsvp successful")
    } catch (err: any) {
      console.error(err);
      alert(err.message);
    } finally {
      setLoading(false)
    }
  }

  async function createEvent(e: any){
    e.preventDefault();
    setLoading(true);
    try {
      console.log("creating event")
      const { value } = await contract.functions.create_event(newEventMax, newEventDeposit, newEventName).txParams({gasPrice: 1}).call();

      console.log("return of create event", value);
      console.log("deposit value", value.deposit.toString());
      console.log("event name", value.name);
      console.log("event capacity", value.max_capacity.toString());
      console.log("eventID", value.unique_id.toString())
      setNewEventID(value.unique_id.toString())
      //setEventId(value.uniqueId.toString())
      setEventCreation(true);
      alert('Event created');
    } catch (err: any) {
      alert(err.message)
    } finally {
      setLoading(false);
    }
  }
return (
  <div className="main">
    <div className="header">Building on Fuel with Sway - Web3RSVP</div>
      <div className="form">
        <h2>Create Your Event Today!</h2>
        <form id="createEventForm" onSubmit={createEvent}>
          <label className="label">Event Name</label>
          <input className="input" value = {newEventName} onChange={e => setNewEventName(e.target.value) }name="eventName" type="text" placeholder="Enter event name" />
          <label className="label">Max Cap</label>
          <input className="input" value = {newEventMax} onChange={e => setNewEventMax(+e.target.value)} name="maxCapacity" type="text" placeholder="Enter max capacity" />
          <label className="label">Deposit</label>
          <input className="input" value = {newEventDeposit} onChange={e => setNewEventDeposit(+e.target.value)} name="price" type="number" placeholder="Enter price" />
          <button className="button" disabled={loading}>
            {loading ? "creating..." : "create"}
          </button>
        </form>
      </div>
      <div className="form">
        <h2>RSVP to an Event</h2>
        <label className="label">Event Id</label>
        <input className="input" name="eventId" onChange={e => setEventId(e.target.value)} placeholder="pass in the eventID"/>
        <button className="button" onClick={rsvpToEvent}>RSVP</button>
      </div>
      <div className="results">
        <div className="card">
          {eventCreation &&
          <>
          <h1> New event created</h1>
          <h2> Event Name: {newEventName} </h2>
          <h2> Event ID: {newEventID}</h2>
          <h2>Max capacity: {newEventMax}</h2>
          <h2>Deposit: {newEventDeposit}</h2>
          <h2>Num of RSVPs: {newEventRSVP}</h2>
          </>
          }
        </div>
          {rsvpConfirmed && <>
          <div className="card">
            <h1>RSVP Confirmed to the following event: {eventName}</h1>
            <p>Num of RSVPs: {numOfRSVPs}</p>
          </div>
          </>}
      </div>
  </div>

);
}
```


# Ejecute su proyecto
Ahora es hora de divertirse, ejecute el proyecto en su navegador.

Desde el interior del directorio `Web3RSVP/frontend `ejecute:

```
$ npm start
Compiled successfully!

You can now view frontend in the browser.

  Local:            http://localhost:3001
  On Your Network:  http://192.168.4.48:3001

Note that the development build is not optimized.
To create a production build, use npm run build.
```
✨⛽✨**¡Felicitaciones! Ha completado su primera DApp en Fuel**✨⛽✨

![](https://i.imgur.com/6ozpQcI.png)

Mencióname en twitter [@camiinthisthang](https://twitter.com/camiinthisthang) y cuéntame que acabas de crear una dapp en Fuel, podrás ser invitado a un grupo privado de creadores, a la siguiente cena de Fuel, obtener acceso Alfa al proyecto o algo más 👀.

**Nota:** Para enviar/hacer push de este proyecto a un repositorio de Github, tebe remover el archivo `.git` que es creado automáticamente al usar el comando `create-react-app`. Puede hacerlo al ejecutar el siguiente comando en `Web3RSVP/frontend`: `Rm -rf .git`. Entonces, ya podrá agregar, hacer commit y enviar estos archivos.

# **Actualizando el contrato**
Si hace cambios a su contrato, aquí están los pasos que debe seguir para mantener sincronizados su interfaz/*frontend* y contrato:

* En su directorio de Contrato, ejecute `forc build`.
* En su directorio de Contrato, vuelva a desplegar el contrato al ejecutar el siguiente comando y siguiendo los mismos pasos de arriba para firmar la transacción con su billetera: `forc deploy --url https://node-beta-1.fuel.network/graphql --gas-price 1`
* En su directorio frontend, vuelva a ejecutar el siguiente comando: `npx fuelchain --target=fuels --out-dir=./src/contracts ../rsvpContract/out/debug/*-abi.json`
* En su directorio frontend, actualice el identificador de contrato (contract ID) en su archivo `App.tsx`

# **¿Necesita ayuda?**
Diríjase al [Discord de Fuel](https://discord.com/invite/fuelnetwork) para obtener ayuda.