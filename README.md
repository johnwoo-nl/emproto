# EM Protocol Handler

Typescript node library to communicate with chargers (aka [EVSEs](https://en.wikipedia.org/wiki/EVSE)) using the "EVSEMaster" app: Besen, Telestar, evseODM, Morec, Deltaco, ...

The library was developed and tested using a Telestar EC311S. Since the other brands use the same app, it may work with them as well,
although there seem to be some subtle differences in supported datagrams and their formats/lengths. Use at your own risk. If something
doesn't work, please set `dumpDatagrams: true` in the createCommunicator `config` parameter in order to see what data is received;
this may help in debugging.

This library is just that: a library, meant for developers to build an app that can communicate with chargers. It implements the
protocol used for communication between the charger and an app, and abstracts away some of the finer implementation specifics.
But, by itself, the library doesn't do much (although there is a small [CLI test runner](#cli-test-runner) included for some basic
testing from the command-line).

This library doesn't do any bluetooth; it is assumed that you have set up a Wi-Fi connection on your charger using the OEM app, and it is
reachable from the host where you run the library. Broadcast UDP packets from the charger should also be available to the library; if you
have placed your charger in a separate network or VLAN, or run the library in a docker container with network separation, then ensure
broadcast datagrams from the charger are routed to the library.
While the OEM app insists on reconnecting via bluetooth regularly, that seems to be an app issue; the charger does in fact remain fully
functional on the network once the Wi-Fi connection is correctly configured.

## Installation

There is no npm package yet; for now, you can use the protocol handler library by cloning the repository and adding a filesystem dependency.

```bash
# Clone repository to local directory.
git clone https://github.com/johnwoo-nl/emproto.git

# Build the library.
cd emproto
npm install && npm run build

# In your own project, add a filesystem dependency to the library project directory.
cd ../my-project
npm install ../emproto
```

Example of a basic app using the library with only a single file, `index.js` (that would go directly in your `my-project` directory as referenced in the above installation):

```javascript
import { createCommunicator } from "emproto";
const evsesFile = '~/evses.json';
(async function() {
    const communicator = createCommunicator();
    communicator.loadEvses(evsesFile);
    await communicator.start();

    communicator.addEventListener(["ADDED", "CHANGED", "REMOVED"], (evse, event) => {
        console.log(`${event} ${evse.toString()}`);
        console.log(`  State: ${JSON.stringify(evse.getState())}`);
        console.log(`  Charge: ${JSON.stringify(evse.getCurrentCharge())}`);
    });

    process.on('SIGINT', () => {
        communicator.stop();
        communicator.saveEvses(evsesFile);
        process.exit();
    });
})();
```

Run your app from a terminal in `my-project`:
```bash
node index.js
```

Use Ctrl+C to exit your app.
This example app doesn't log in to any chargers so it'll show only basic info (no state info), unless another app has saved the password to `~/evses.json`.
Read on to see how to login and get more info from a charger, and how to control it.

## Requirements

- Nodejs 20.14.10 or newer.
- Build toolchain or runtime supporting ES6 modules.
- Typescript to build (`tsc`), for now. Once an npm package will be created with prebuilt JS, JS will also suffice on the app side.

## Library usage

### Communicator

The `EmCommunicator` interface (implemented by an internal `Communicator` class) is the entry point to the library, keeping track of EVSE instances and communicating over the network.
You can create a communicator instance using the `createCommunicator` factory function exported by the main library index file.

```typescript
import { createCommunicator } from "emproto";

// Instantiate a communicator.
const communicator = createCommunicator();  // communicator instanceof EmCommunicator

// Listen for events.
communicator.addEventListener(["ADDED", "CHANGED", "REMOVED"], (evse, event) => {
    // EVSE was added, changed or removed (event in second parameter).
    // Maybe your app wants to update its UI or perform some action now.
    // You could also register separate listeners for separate events. 
});

// You can remove event listeners again, but if your app just creates a single
// communicator instance for the duration of its lifetime, you could skip this.
// The listener must be the same Function instance that you passed to
// addEventListener.
communicator.removeEventListener("CHANGED", listener);

// If you wish to persist the EVSEs list, you can load it from a file (or array
// previously obtained from getEvses). This is optional; you could also have the
// communicator re-discover the EVSEs on next run, but you'd have to login again
// (your app would need to save the password somewhere). By saving the EVSEs list
// and loading it here, the communicator will automatically login to the EVSEs
// again and keep their state up-to-date.
communicator.loadEvses("~/evses.json");

// A communicator can always hold EVSEs state, but won't start communicating with
// EVSEs until it is started. This method may throw an error if there is some issue
// setting up the UDP socket.
await communicator.start();

// Get a list of EVSEs currently in the communicator's memory.
communicator.getEvses().forEach(evse => {
    // Do something with the EVSE.
    // evse instanceof EmEvse
});

// Or get a specific EVSE by serial (which is the unique identifier of an EVSE).
// Returns undefined if there is no EVSE with given serial.
const evse = communicator.getEvse('1234567890'); // evse instanceof EmEvse

// When your app is done, stop the communicator.
communicator.stop();

// You may also wish to persist the EVSEs list when your app is stopping.
communicator.saveEvses("~/evses.json");
```

### EVSE

The `EmEvse` interface (implemented by an internal class `Evse`) represents a single charger and exposes the functionality to read and interact with it.
You can obtain `EmEvse` instances from the communicator.

#### Getting EVSE basic info

```typescript
import { EmEvseInfo, EmEvseConfig } from "emproto/types";

// Get the general info of an EVSE (which you got from the communicator via getEvse,
// getEvses or an event listener).
const info = evse.getInfo();

// Info is an EmEvseInfo instance.
console.log(`Serial: ${info.serial}`);
console.log(`Brand and model: ${info.brand} ${info.model}`);
console.log(`Software version: ${info.softwareVersion}`);
console.log(`Maximum total supported output power: ${info.maxPower} watts`);
console.log(`Maximum supported output current per phase: ${info.maxElectricity} amps`);
```

#### Logging in

```typescript
// Use the same 6-digit password as for the OEM app.
// This method will throw an error if the password is incorrect.
await evse.login("123456");

// Note: upon successful login, the communicator will request the EVSE's configuration.
// The password is kept in the EVSE instance and is persisted via saveEvses/loadEvses.
// This allows the communicator to automatically login to the EVSE again when it is 
// started again at a later time.
```

#### Reading EVSE status

Once logged in, the EVSE will start sending us status info periodically.  After logging in,
it can take a few seconds before the first update arrives, and the state will be  updated
about once every 5-10 seconds. The library checks incoming info against the current state
it has, and if anything changed, the state will be updated and a "changed" event will be
fired for the EVSE.

```typescript
import { EmEvseState } from "emproto/types";

const state = evse.getState();  // state instanceof EmEvseState

// State fields.
console.log(`State: ${state.currentState}`); // See EmEvseCurrentState enum
console.log(`Gun state: ${state.gunState}`); // See EmEvseGunState enum
console.log(`Charging state: ${state.outputState}`); // See EmEvseOutputState enum

// Meta state: what things an app can effectively do with an EVSE depends on the above three
// protocol-level states. However most apps won't be interested in all protocol-level state
// detail (doing many switch cases or if-else branches), and will just want to know the overall
// state of the EVSE, e.g. to display their UI or know if they can start a charge. For this
// common use case, the library exposes a meta state (computed from the above states) that is
// a single enum which gives the overall state, abstracting away the protocol-level states.
// It is expected and intended that most apps will only ever need this meta state and won't
// need to check the separate protocol-level state fields. See EmEvseMetaState enum for
// possible values.
console.log(`Meta state: ${evse.getMetaState()}`);
```

#### Electricity metering

The `EmEvseState` structure offers the current power output (across all phases) in watts, as well
as current voltages and amps per phase.

```typescript
import { EmEvseState } from "emproto/types";

const state = evse.getState();  // state instanceof EmEvseState

// Power across all phases.
console.log(`Current power: ${state.currentPower} W`);
// Volts and amps per phase (l2 and l3 also available).
console.log(`Current voltage (phase L1): ${state.l1Voltage} V`);
console.log(`Current amps (phase L1): ${state.l1Electricity} A`);
```

#### Temperature

The protocol specifies two temperature values: inner and outer.
On my Telestar, both values are set to the same value, which is the inner temperature. The OEM app also seems to
only ever use (show) the inner temperature. Expect the inner temperature to always be set; the outer temperature
could be empty, or set to the same value as the inner temperature.

The values of these fields are always in degrees Celsius; your app can consult the `getConfig().temperatureUnit`
field to check if that is the user-preferred unit, or convert to Fahrenheit for display if needed.

```typescript
const state = evse.getState();  // state instanceof EmEvseState

console.log(`Inner temperature: ${state.innerTemp} °C`);
console.log(`Outer temperature: ${state.outerTemp} °C`);
```

#### EVSE errors

> [!NOTE]  
> The protocol specifies a 32-bit integer field for errors. The OEM app has a funny way to decode errors from this
> field: it converts the number to a binary string (of 32 zeroes and ones) and then finds the index of the first '1'
> character. And judging by the error values (mapping to a bit position in this field), it seems that this field is
> actually a bitfield that could have more than one error flag set simultaneously. The OEM app's way of decoding via
> a binary string is not just funny as the same index could be found more easily using bitwise arithmetic, but
> finding the index of the first '1' character also means that it will only ever detect the first set error,
> ignoring the rest of the bits in the 32-bit integer field.

The library instead will test each of the 32 bits and will return an array of error codes (`EmEvseErrorState` values),
one for each set bit. You can test for errors by checking whether the array isn't empty (or use the meta state which
will return `ERROR` if at least one error is set, i.e. the array is not empty).

```typescript
// Errors list as an array of EmEvseErrorState values. May this array forever remain empty.
console.log(`Errors: ${state.errors.length === 0 ? "none!" : state.errors.join(", ")}`);
```

#### Getting configuration

`EmEvse.getConfig()` returns the current configuration. It is only available when the library has logged in at
least once, and is not async and non-blocking, just returning whatever state is currently in memory.

```typescript
// getConfig returns the current config. It is not async and non-blocking.
const config = evse.getConfig();

// Config is an EmEvseConfig instance.
console.log(`Name: ${config.name}`);
console.log(`Temperature unit: ${config.temperatureUnit}`);
console.log(`Language: ${config.language}`);
console.log(`Offline charging: ${config.offLineCharge}`);
console.log(`Configured maximum current per phase: ${config.maxElectricity}`);
// Note: config.maxElectricity is not actually used directly by the EVSE for charging
// sessions; it seems to just be a way to persist the user's preference on the EVSE so
// that various apps can share it. The actual current limit for charging is passed as a
// parameter to chargeStart. Your app should default to this configured maxElectricity
// when presenting the user with options to start a charging session. It should also
// update this configuration field if the user changed the current limit when starting
// a session, so other apps will know about it and can also use that limit.
```

Immediately after logging in, the communicator will request the config to keep itself up-to-date. Since `login()`
will have returned already, not all fields may be immediately set. Or, if the communicator logged in to the EVSE
before, there may be a stale value. If you want to be certain that you have the current, live config, use
`fetchConfig`. This async method returns a promise that will resolve with the config once it's available. You
can call this method right after `login()` returns (it will recycle the same promise if the config is already being
fetched at the time of the call). By default, an existing config is returned without going to the charger if it was
just fetched (within the last 5 seconds). This max-age (in seconds) can be specified as an argument to `fetchConfig`.

```typescript
const freshConfig = await evse.fetchConfig();
console.log(`Name: ${freshConfig.name}`);
```

You can also just initiate fetchConfig without waiting for its return value to trigger an update. Any changes
in config will also result in a `changed` event for the EVSE. This is useful if you wish to add some "Refresh"
button to your app's UI of the EVSE's configuration, and your UI is driven by these events.

```typescript
// Never return any in-memory, potentially stale config; always go to the charger for a fresh update.
// May throw an error if there's some communication issue.
evse.fetchConfig(0).then();
```

#### Changing configuration

The EVSE must be online and logged in to change these configuration settings. Upon successful change, the EVSE's
config data structure will also be updated and a `changed` event will be emitted for the EVSE.

```typescript
await evse.setName("My charger");
await evse.setOffLineCharge(OffLineChargeStatuses.DISABLED);
await evse.setTemperatureUnit(TemperatureUnits.CELSIUS);
await evse.setLanguage(Languages.ENGLISH);
```

#### Starting a charging session

```typescript
import { ChargeStartParams } from "emproto/types";

// Start a charging session using 6 amps. If this value is different from the one
// configured for the EVSE (in getConfig().maxElectricity), this new value will be
// written to the EVSE so other apps will also have the updated amps value.
await evse.chargeStart({ maxAmps: 6 });

// You can omit the maxAmps parameter to use the currently configured value from
// getConfig().maxElectricity.
await evse.chargeStart();

// You may also specify a user name of the person using your app. This name will be
// visible in historical charge records (including those in the OEM app). Maximum
// length is 16 ASCII characters.
await evse.chargeStart({ userId: "John Doe" });

// You may also specify a custom identifier for the session, for example to correlate
// the session stored on the EVSE with one stored in your app's database. Maximum
// length is 16 ASCII characters.
await evse.chargeStart({ chargeId: "ABC123" });

// If you have a 3-phase EVSE, by default the EVSE will use all 3 phases for charging.
// You can limit this to single phase:
await evse.chargeStart({ singlePhase: true });

// You can delay-start a session by specifying a start time. If omitted or the time
// is not in the future, charging will start immediately. Note that there is a limit
// to how far into the future you can plan a charge session (24 hours on my Telestar);
// chargeStart will throw an error if you set a too distant time. Also note that the
// car must be plugged in before calling chargeStart even for planned sessions; if the
// car isn't plugged in, chargeStart will throw an error. Starting one hour from now:
await evse.chargeStart({
    maxAmps: 6,
    startAt: new Date(Date.now() + (3600 * 1000))
});

// You can limit the duration (in minutes, integer) and/or the amount of energy (in
// kWh, float) of the session:
await evse.chargeStart({
    maxDurationMinutes: 90,
    maxEnergyKWh: 7.5
});
// Note: maxEnergyKWh doesn't appear to work yet (always shows unlimited in the OEM app,
//       although it also doesn't seem to work when starting a session in OEM app itself).
```

#### Getting info about the current charging session

```typescript
import { EmEvseCurrentCharge } from "emproto/types";

// If the EVSE is currently charging, this returns info about the ongoing charging session.
// If it's not charging but a session is planned, then this returns info about the planned
// session. If there is no session ongoing or planned, this returns info about the last finished
// session. Otherwise, it returns undefined.
// Note that you can use chargeStop() both to stop an ongoing session and to cancel a planned
// session.
const currentCharge = evse.getCurrentCharge();

// How many kWh have been charged in this session.
console.log(`Charged energy: ${currentCharge.chargeKWh} kWh`);

// How long a current session is going on, or (if finished) how long it took.
console.log(`Duration: ${currentCharge.durationSeconds} seconds`);

// When the session was entered into the EVSE. If it's not a planned session (currentState
// is not CHARGING_RESERVATION), then the session will also have started at this time.
console.log(`Start time: ${currentCharge.startDate.toISOString()}`);

// When the session will start charging (for reservations/planned sessions, if currentState
// is CHARGING_RESERVATION).
console.log(`Start time: ${currentCharge.reservationDate.toISOString()}`);

// The maximum current used for this session.
console.log(`Max current: ${currentCharge.maxElectricity} A`);

// The maximum duration for this session in minutes, as specified at start/reservation time.
console.log(`Max duration: ${currentCharge.maxDurationMinutes} minutes`);

// The maximum energy to charge for this session in kWh, as specified at start/reservation time.
console.log(`Max energy: ${currentCharge.maxEnergyKWh} kWh`);
```

#### Stopping a charging session

```typescript
// Stop a charging session.
await evse.chargeStop();

// Like when starting a session, you can also specify a user name of the person who
// stopped the session:
await evse.chargeStop({ userId: "John Doe" });
````

Note: `chargeStop` will also cancel a planned charging session, if one was set using `startAt` as an option for `chargeStart`.

## CLI test runner

The CLI runner is also written in Typescript, so a plain Node.js runtime will not be able to run it directly.
You can run it from the emproto root like this:

```bash
npx tsx clitest
```
This will discover EVSEs on the network and print some info as changes are detected. Press Ctrl+C to exit.

Add `dump` (to any command) to dump incoming and outgoing datagrams (note: once logged in, you'll get more info but the EVSE's password will be present in the dumped datagrams, so don't copy-paste them to the internet).

To log in, set a password like this:
```bash
npx tsx clitest EC311S=123456
```
"EC311S" in this example is the model name of the charger, but you can use any part of the serial, brand or model. The CLI runner will try to log in using the 6-digit password on the first matching EVSE.
If login is successful, the password will also be persisted in ~/evses.json, so you can just run `npx tsx clitest` next time and see the EVSE's detail info without needing to specify the password again.
For EVSEs that the CLI runner can log in to, it will also print the state and the current charging session info.

To show info of a specific EVSE, use the same type of filter as for login:
```bash
npx tsx clitest EC311S
```

To start a session, use the `start=<amps>` command, specifying the maximum amperage to use. You can specify a filter to run the command on a specific EVSE; otherwise the first online EVSE that the CLI runner can log in to will be used.
```bash
npx tsx clitest start=6

# or to target the same EVSE as the examples above:

npx tsx clitest EC311S start=6
```

To stop a session, use the `stop` command. You can again specify a filter to target a specific EVSE (instead of using the first available online one).
```bash
npx tsx clitest stop
```


# IMPORTANT NOTE

The CLI runner makes it easy to quickly run start/stop commands. But each start c.q. stop will
cause both the EVSE's AC phase relays and the car's high-voltage DC contactors to engage c.q. disengage.
Doing this too often in quick succession **will wear these parts**!

The same goes for the library's `chargeStart` and `chargeStop` methods; your app should block excessive
starts and stops in short timespans.

Note that exact behavior may differ by car; some cars leave their contactors engaged for a short while (30 seconds
to a minute) after a session is stopped **normally** (using `chargeStop`, with the CP pin still connected -- not
when unplugged hot). For such cars, starting a new session before the contactors have disengaged will not cause wear,
and your app could start a new session (e.g. using a different amperage for solar charging) immediately. It will
probably still take a minute or so for that new session to ramp up to maximum current. I do not have a list of cars
with their actual behavior (which may vary by model year or even software version), you'd have to figure that out
yourself. In some cases the (dis)engagement of the contactors can be audible.
