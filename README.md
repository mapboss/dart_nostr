# Dart Nostr

This is a Dart/Flutter toolkit for developing [Nostr](https://nostr.com/) client apps faster and easier.

## Table of Contents

- [Installation](#installation)
- [Usage](#usage)
  - [Singleton instance vs. multiple instances](#singleton-instance-vs-multiple-instances)
  - [Keys](#keys)
    - [Generate private and public keys](#generate-private-and-public-keys)
    - [Generate a key pair from a private key](#generate-a-key-pair-from-a-private-key)
    - [Sign & verify with a private key](#sign--verify-with-a-private-key)
    - [More functionalities](#more-functionalities)
  - [Events & Relays](#events--relays)

## Installation

Install the package by adding the following to your `pubspec.yaml` file:

```yaml
dependencies:
  dart_nostr: any
```

Otherwise you can install it from the command line:

```bash
# Flutter project
flutter pub add dart_nostr

# Dart project
dart pub add dart_nostr
```

## Usage

### Singleton instance vs. multiple instances

The base and only class you need to remember is `Nostr`, all methods and utilities are available through it.

The `Nostr` class is accessible through two ways, a singleton instance which you can access by calling `Nostr.instance` and a constructor which you can use to create multiple instances of `Nostr`.

Each instance (including the singleton instance) is independent from each other, so everything you do with one instance will be accessible only through that instance including relays, events, caches, callbacks, etc.

Use the singleton instance if you want to use the same instance across your app, so as example once you do connect to a set of relays, you can access and use them (send and receive events) from anywhere in your app.

Use the constructor if you want to create multiple instances of `Nostr`, as example if you want to connect to different relays in different parts of your app, or if you have extensive Nostr relays usage (requests) and you want to separate them into different instances so you avoid relays limits.

```dart
/// Singleton instance
final instance = Nostr.instance;

/// Constructor
final instance = Nostr();
```

### Keys

#### Generate private and public keys

```dart
final newKeyPair = instance.keysService.generateKeyPair();

print(newKeyPair.public); // Public key
print(newKeyPair.private); // Private key
```

#### Generate a key pair from a private key

```dart
final somePrivateKey = "HERE IS MY PRIVATE KEY";

final newKeyPair = instance.keysService
      .generateKeyPairFromExistingPrivateKey(somePrivateKey);

print(somePrivateKey == newKeyPair.private); // true
print(newKeyPair.public); // Public key
```

#### Sign & verify with a private key

```dart

/// sign a message with a private key
final signature = instance.keysService.sign(
  privateKey: keyPair.private,
  message: "hello world",
);

print(signature);

/// verify a message with a public key
final verified = instance.keysService.verify(
  publicKey: keyPair.public,
  message: "hello world",
  signature: signature,
);

print(verified); // true
```

Note: `dart_nostr` provides even more easier way to create, sign and verify Nostr events, see the relays and events sections below.

#### More functionalities

The package exposes more useful methods, like:
  
```dart
// work with nsec keys
instance.keysService.encodePrivateKeyToNsec(privateKey);
instance.keysService.decodeNsecKeyToPrivateKey(privateKey);

// work with npub keys
instance.keysService.encodePublicKeyToNpub(privateKey);
instance.keysService.decodeNpubKeyToPublicKey(privateKey);

// more keys derivations and validations methods
instance.keysService.derivePublicKey(privateKey);
instance.keysService.generatePrivateKey(privateKey);
instance.keysService.isValidPrivateKey(privateKey);

// general utilities that related to keys
instance.utilsService.decodeBech32(bech32String);
instance.utilsService.encodeBech32(bech32String);
instance.utilsService.pubKeyFromIdentifierNip05(bech32String);
```

### Events & Relays

#### Create an event

Quickest way to create an event is by using the `NostrEvent.fromPartialData` constructor, it does all the heavy lifting for you, like signing the event with the provided private key, generating the event id, etc.

```dart
final event = NostrEvent.fromPartialData(
  kind: 1,
  content: 'example content',
  keyPairs: keyPair, // will be used to sign the event
  tags: [
    ['t', currentDateInMsAsString],
  ],
);

print(event.id); // event id
print(event.sig); // event signature

print(event.serialized()); // event as serialized JSON
```

Note: you can also create an event from scratch with the `NostrEvent` constructor, but you will need to do the heavy lifting yourself, like signing the event, generating the event id, etc.

#### Connect to relays
for a single `Nostr` instance, you can connect and reconnect to multiple relays once or multiple times, so you will be able to send and receive events later.

```dart
try {
 
 final relays = ['wss://relay.damus.io'];
 
 await instance.relaysService.init(
  relaysUrl: relays,
 );

print("connected successfully")
} catch (e) {
  print(e);
}
```

if anything goes wrong, you will get an exception with the error message.
Note: the `init` method is highly configurable, so you can control the behavior of the connection, like the number of retries, the timeout, wether to throw an exception or not, register callbacks for connections or events...

#### Listen to events

##### As a stream 

```dart
```dart
// Creating a request to be sent to the relays. (as example this request will get all events with kind 1 of the provided public key)
final request = NostrRequest(
  filters: [
    NostrFilter(
      kinds: const [1],
      authors: [keyPair.public],
    ),
  ],
);


// Starting the subscription and listening to events
final nostrStream = Nostr.instance.relaysService.startEventsSubscription(
  request: request,
  onEose: (ease) => print(ease),
);

print(nostrStream.subscriptionId); // The subscription id

// Listening to events
nostrStream.stream.listen((NostrEvent event) {
  print(event.content);
});

// close the subscription later
nostrStream.close();
```

##### As a future (resolves on EOSE)

```dart
// Creating a request to be sent to the relays. (as example this request will get all events with kind 1 of the provided public key)
final request = NostrRequest(
  filters: [
    NostrFilter(
      kinds: const [1],
      authors: [keyPair.public],
    ),
  ],
);

// Call the async method and wait for the result
final events =
    await Nostr.instance.relaysService.startEventsSubscriptionAsync(
  request: request,
);

// print the events
print(events.map((e) => e.content));
```

Note: `startEventsSubscriptionAsync` will be resolve with an `List<NostrEvent>` as soon as a relay sends an EOSE command.

#### Reconnect & disconnect

```dart
// reconnect
await Nostr.instance.relaysService.reconnectToRelays(
       onRelayListening: onRelayListening,
       onRelayConnectionError: onRelayConnectionError,
       onRelayConnectionDone: onRelayConnectionDone,
       retryOnError: retryOnError,
       retryOnClose: retryOnClose,
       shouldReconnectToRelayOnNotice: shouldReconnectToRelayOnNotice,
       connectionTimeout: connectionTimeout,
       ignoreConnectionException: ignoreConnectionException,
       lazyListeningToRelays: lazyListeningToRelays,
     );
    
// disconnect
await Nostr.instance.relaysService.disconnectFromRelays();
```

#### Send an event

```dart
// sending synchronously
Nostr.instance.relaysService.sendEventToRelays(
  event,
  onOk: (ok) => print(ok),
);

// sending synchronously with a custom timeout
final okCommand = await Nostr.instance.relaysService.sendEventToRelaysAsync(
  event,
  timeout: const Duration(seconds: 3),
);

print(okCommand);
```

Note: `sendEventToRelaysAsync` will be resolve with an `OkCommand` as soon as one relay accepts the event.

#### Send NIP45 COUNT

```dart
// create a count event
final countEvent = NostrCountEvent.fromPartialData(
  eventsFilter: NostrFilter(
    kinds: const [0],
    authors: [keyPair.public],
  ),
);

// Send the count event synchronously
Nostr.instance.relaysService.sendCountEventToRelays(
  countEvent,
  onCountResponse: (countRes) {
    print('count: $countRes');
  },
);

// Send the count event asynchronously
final countRes = await Nostr.instance.relaysService.sendCountEventToRelaysAsync(
  countEvent,
  timeout: const Duration(seconds: 3),
);

print("found ${countRes.count} events");
```

#### Relay Metadata NIP11

```dart
final relayDoc = await Nostr.instance.relaysService.relayInformationsDocumentNip11(
  relayUrl: "wss://relay.damus.io",
);

print(relayDoc?.name);
print(relayDoc?.description);
print(relayDoc?.contact);
print(relayDoc?.pubkey);
print(relayDoc?.software);
print(relayDoc?.supportedNips);
print(relayDoc?.version);
```

#### More functionalities

The package exposes more useful methods, like:

```dart
// work with nevent and nevent
final nevent = Nostr.instance.utilsService.encodeNevent(
  eventId: event.id,
  pubkey: pubkey,
  userRelays: [],
);
  
print(nevent);

final map = Nostr.instance.utilsService.decodeNeventToMap(nevent);
print(map);


// work with nprofile
final nprofile = Nostr.instance.utilsService.encodeNProfile(
  pubkey: pubkey,
  userRelays: [],
);

print(nprofile);

final map = Nostr.instance.utilsService.decodeNprofileToMap(nprofile);
print(map);

```

### More utils

#### Generate random 64 hex

```dart
final random = Nostr.instance.utilsService.random64HexChars();
final randomButBasedOnInput = Nostr.instance.utilsService.consistent64HexChars("input");

print(random);
print(randomButBasedOnInput);
```

#### NIP05 related

```dart
/// verify a nip05 identifier
final verified = await Nostr.instance.utilsService.verifyNip05(
  internetIdentifier: "something@domain.com",
  pubKey: pubKey,
);

print(verified); // true

  
/// Validate a nip05 identifier format
final isValid = Nostr.instance.utilsService.isValidNip05Identifier("work@gwhyyy.com");
print(isValid); // true

/// Get the pubKey from a nip05 identifier
final pubKey = await Nostr.instance.utilsService.pubKeyFromIdentifierNip05(
  internetIdentifier: "something@somain.c",
);
  
print(pubKey);
```

### NIP13 hex difficulty

```dart
Nostr.instance.utilsService.countDifficultyOfHex("002f");
```


### Other