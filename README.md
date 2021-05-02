# aws-credstash

> Use [credstash](https://github.com/fugue/credstash) in NodeJS

Drop-in replacement for [nodecredstash](https://www.npmjs.com/package/nodecredstash), API fully compatible (largely unmodified fork of nodecredstash).

## Why this fork:

- Allows your module to enforce the version of `aws-sdk`: is now a peer dependency
- Types are updated by package maintainer

See **[Release Changelog](https://github.com/tongrhj/aws-credstash/releases)** for more information

## Installation

```
$ npm install aws-credstash
```

## Usage

Store and retrieve a secret:

```js
import Credstash from "aws-credstash";

const credstash = new Credstash({
  table: "credential-store",
  awsOpts: { region: "us-west-2" },
});

(async () => {
  const name = "Death Star vulnerability";
  const context = { app: "db", enviroment: "production" };
  const version = await credstash.incrementVersion({ name });
  await credstash.putSecret({ name, secret: "Exhaust vent", version, context });
  console.log("Secret updated!");

  const secret = await credstash.getSecret({ name, context });
  console.log(secret); // 'Exhaust vent'
})();
```

## API

### Credstash(CredstashConfig)

Returns a new `credstash` instance

#### CredstashConfig

Type: `object`

#### table

Type: `string` \
Default: `credential-store`

The DynamoDB table to store credentials

#### kmsKey

Type: `string` \
Default: `alias/credstash`

The name of the [KMS key](http://docs.aws.amazon.com/kms/latest/developerguide/create-keys.html) created for credstash.

#### awsOpts

Type: `AWS.KMS.ClientConfiguration`

Options to be passed to the [aws-sdk](http://docs.aws.amazon.com/AWSJavaScriptSDK/guide/node-services.html) instance for DynamoDB and KMS
Specifig configurations can be found for [DynamoDB](http://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/DynamoDB.html#constructor-property) and [KMS](http://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/KMS.html#constructor-property).
`region` can be sent in as a parameter, or you can follow other [AWS conventions](http://docs.aws.amazon.com/AWSJavaScriptSDK/guide/node-configuring.html) for setting the region

Example:

```
{
  "region": "us-east-1"
}
```

#### [dynamoOpts](http://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/DynamoDB.html#constructor-property)

Options that are specific to the DynamoDB configuration. Defaults can still be assigned in awsOpts, but they can be overridden just for dynamoDb here

#### [kmsOpts](http://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/KMS.html#constructor-property)

Options that are specific to the KMS configuration. Defaults can still be assigned in awsOpts, but they can be overridden just for kms here

### Common Options

#### name

The name of the secret that will be stored in DynamoDB

#### [version](https://github.com/fugue/credstash#versioning-secrets)

Can be a string or number. If it is a number, then aws-credstash will pad it with 0s so it can be sorted.

#### [context](http://docs.aws.amazon.com/kms/latest/developerguide/encryption-context.html)

Used to get the encryption key from KMS.

Type: `object`

#### callback

An optional callback function when you don't want to use promises;

```js
credstash.getSecret(
  {
    name: "Death Star plans",
    context: { rebelShip: "true" },
  },
  function (err, res) {
    if (err) {
      throw new Error("The Death Star plans are not in the main computer.");
    }
  }
);
```

### credstash

`Credstash` instance

#### .createDdbTable([cb])

Create the table in DynamoDB using the [table](table) option

#### .putSecret({name, secret, [version], [context]}, [cb])

Encode a secret and place it in DynamoDB.

```js
credstash.putSecret({
  name: "Death Star Vulnerability",
  secret: "Exhaust vent",
  context: { rebel: "true" },
});
```

DynamoDB will now contain a record for this entry that looks like:

```js
{
  "name": "Death Star Vulnerability", //
  "key": "...", // The value sent to KMS to retrieve the decryption key
  "version": "0000000000000000001", // The version string, should be sorteable
  "hmac": "...", // An HMAC validation value
  "contents": "..." // The AES 128 encrypted value
}
```

#### getHighestVersion({name}, [cb])

Returns the first sorted result for the given name key.

#### incrementVersion({name}, [cb])

Returns the next incremented version version for the given name key.

#### .getSecret({ name, [version], [context] }, [cb])

Retrieve a decrypted secret from DynamoDB

#### .getAllSecrets({[version], [context], [startsWith]}, [cb])

Retrieve all decrypted secrets from DynamoDB.

The **startsWith** option will filter the response

```js
credstash
  .getAllSecrets({ context: { rebel: "true" } })
  .then((secrets) => console.log(JSON.stringify(secrets, null, 2)));
```

```js
{
  "Death Star vulnerability": "Exhaust vent"
}
```

#### .getAllVersions({name, [limit]}, [cb])

Retrieve all or the last N(limit) versions of a secret.

```js
credstash
  .getAllVersions({
    name: "Death Star vulnerability",
    limit: 2,
    context: { rebel: "true" },
  })
  .then((secrets) => console.log(JSON.stringify(secrets, null, 2)));
```

```js
[
  { version: "0000000000000000006", secret: "Exhaust vent" },
  { version: "0000000000000000005", secret: "Destroy vent" },
];
```

#### .listSecrets([cb])

Retrieve all stored secrets and their highest version

```js
credstash
  .listSecrets()
  .then((list) => console.log(JSON.stringify(list, null, 2)));
```

```js
[
  {
    name: "Death Star",
    version: "0000000000000000001",
  },
  {
    name: "Death Star vulnerability",
    version: "0000000000000000001",
  },
];
```

#### .deleteSecret({name, version}, [cb])

Delete the desired secret by version from DynamoDB

```js

credstash.deleteSecret({name: 'Death Star', version: 1})
// 'Deleting Death Star -- version 0000000000000000001'
  .then(() => credstash.list())
  .then(list => console.log(JSON.stringify(list, null, 2));
```

```js
[
  {
    name: "Death Star vulnerability",
    version: "0000000000000000001",
  },
];
```

#### .deleteSecrets(name)

Deletes all of the versions of `name`

```js

credstash.deleteSecrets({name: 'Death Star vulnerability'})
// 'Deleting Death Star vulnerability -- version 0000000000000000001'
  .then(() => credstash.listSecrets())
  .then(list => console.log(JSON.stringify(list, null, 2));
```

```js
[];
```
