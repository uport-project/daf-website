---
id: node_setup_identifiers
title: Node Setup & Identifiers
sidebar_label: Setup & Identifiers
---

This guide covers setting up an agent and creating identifiers in Node.

### Prerequisites

You need to have Node v10 or later installed. In this example, we use yarn as the package manager, but you can also use npm.

Start by creating a directory for our project and initializing the npm package.

```bash
mkdir veramo-agent && cd veramo-agent
yarn init -y
```

Install dev dependencies

```bash
yarn add typescript ts-node --dev
```

Install Veramo core and plugins

```bash
yarn add @veramo/core @veramo/credential-w3c @veramo/data-store @veramo/did-manager @veramo/did-provider-ethr @veramo/did-provider-web @veramo/did-resolver @veramo/key-manager @veramo/kms-local ethr-did-resolver web-did-resolver
```

Install `sqlite`

```bash
yarn add sqlite3
```

Add a tsconfig.json to your project

```json
{
  "compilerOptions": {
    "preserveConstEnums": true,
    "strict": true,
    "target": "es6",
    "rootDir": "./",
    "moduleResolution": "node",
    "esModuleInterop": true,
    "downlevelIteration": true
  }
}
```

## Bootstrap Veramo

We bootstrap Veramo by creating a setup file and initializing the agent. Create a setup file in `src/veramo/setup.ts` and import the following dependencies:

```ts
// Core interfaces
import { createAgent, IDIDManager, IResolver, IDataStore, IKeyManager } from '@veramo/core'

// Core identity manager plugin
import { DIDManager } from '@veramo/did-manager'

// Ethr did identity provider
import { EthrDIDProvider } from '@veramo/did-provider-ethr'

// Web did identity provider
import { WebDIDProvider } from '@veramo/did-provider-web'

// Core key manager plugin
import { KeyManager } from '@veramo/key-manager'

// Custom key management system for RN
import { KeyManagementSystem } from '@veramo/kms-local'

// Custom resolvers
import { DIDResolverPlugin } from '@veramo/did-resolver'
import { Resolver } from 'did-resolver'
import { getResolver as ethrDidResolver } from 'ethr-did-resolver'
import { getResolver as webDidResolver } from 'web-did-resolver'

// Storage plugin using TypeOrm
import { Entities, KeyStore, DIDStore, IDataStoreORM } from '@veramo/data-store'

// TypeORM is installed with daf-typeorm
import { createConnection } from 'typeorm'

// SQLite setup
import { Initial1616938713828 } from '../migration/1616938713828-initial'
```

Create some variables that we will use later

```ts
// This will be the name for the local sqlite database for demo purposes
const DATABASE_FILE = 'database.sqlite'

// You will need to get a project ID from infura https://www.infura.io
const INFURA_PROJECT_ID = 'INFURA_PROJECT_ID'
```

Initialise a database using TypeORM

```ts
const dbConnection = createConnection({
  type: 'sqlite',
  database: DATABASE_FILE,
  migrations: [ Initial1616938713828 ],
  migrationsRun: true,
  logging: ['error', 'info', 'warn'],
  entities: Entities,
})
```

Create the agent by using the createAgent method from `@veramo/core`

```ts
export const agent = createAgent<IDIDManager & IKeyManager & IDataStore & IDataStoreORM & IResolver>({
  plugins: [
    new KeyManager({
      store: new KeyStore(dbConnection),
      kms: {
        local: new KeyManagementSystem(),
      },
    }),
    new DIDManager({
      store: new DIDStore(dbConnection),
      defaultProvider: 'did:ethr:rinkeby',
      providers: {
        'did:ethr:rinkeby': new EthrDIDProvider({
          defaultKms: 'local',
          network: 'rinkeby',
          rpcUrl: 'https://rinkeby.infura.io/v3/' + INFURA_PROJECT_ID,
        }),
        'did:web': new WebDIDProvider({
          defaultKms: 'local',
        }),
      },
    }),
    new DIDResolverPlugin({
      resolver: new Resolver({
        ethr: ethrDidResolver({
          networks: [{ name: 'rinkeby', rpcUrl: 'https://rinkeby.infura.io/v3/' + INFURA_PROJECT_ID }],
        }).ethr,
        web: webDidResolver().web,
      }),
    }),
  ],
})
```

Then create a migrations file in `src/migration/1616938713828-initial.ts` and copy [this file](https://raw.githubusercontent.com/uport-project/veramo-website/master/docs/resources/1616938713828-initial.ts) into it.

That's the minimal agent setup complete. Let's use it to create and list identifiers.

## App Logic

Create 2 files `./src/create-identifier.ts` and `./src/list-identifiers.ts`

Add the following code to `./src/list-identifiers.ts`

```ts
import { agent } from './veramo/setup'

async function main() {
  const identifiers = await agent.didManagerFind()

  console.log(`There are ${identifiers.length} identifiers`)

  if (identifiers.length > 0) {
    identifiers.map((id) => {
      console.log(id)
      console.log('..................')
    })
  }
}

main().catch(console.log)
```

Add the following code to `./src/create-identifier.ts`

```ts
import { agent } from './veramo/setup'

async function main() {
  const identity = await agent.didManagerCreate()
  console.log(`New identity created`)
  console.log(identity)
}

main().catch(console.log)
```

To run those functions add the following script commands to `package.json`

```json
{
  "scripts": {
    "id:list": "ts-node ./src/list-identifiers",
    "id:create": "ts-node ./src/create-identifier"
  }
}
```

### List Identifiers

```bash
yarn id:list
```

**Expected output**

```bash
$ ts-node ./src/list-identifiers
Please provide SecretBox to the KeyStore
There are 0 identifiers
```

### Create Identifier

```bash
yarn id:create
```

**Expected output**

```bash
$ ts-node ./src/create-identifier
Please provide SecretBox to the KeyStore
New identity created
{ did:
   'did:ethr:rinkeby:0x6acf3bb1ef0ee84559de2bc2bd9d91532062a730',
  controllerKeyId:
   '04f3f9457f21af89ce4209f0c8c07f04cb14b3420e5a598b96c564127f693687f6273ed52896c91d59b443f260a34e37742f8f35a1a6beafa08ccbc66df363bd44',
  keys:
   [ { type: 'Secp256k1',
       kid:
        '04f3f9457f21af89ce4209f0c8c07f04cb14b3420e5a598b96c564127f693687f6273ed52896c91d59b443f260a34e37742f8f35a1a6beafa08ccbc66df363bd44',
       publicKeyHex:
        '04f3f9457f21af89ce4209f0c8c07f04cb14b3420e5a598b96c564127f693687f6273ed52896c91d59b443f260a34e37742f8f35a1a6beafa08ccbc66df363bd44',
       kms: 'local' } ],
  services: [],
  provider: 'did:ethr:rinkeby' }
```

Congrats, You have set up the agent and created identifiers!
