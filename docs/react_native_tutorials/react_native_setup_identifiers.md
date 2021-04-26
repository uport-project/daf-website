---
id: react_native_setup_identifers
title: React Native Setup & Identifiers
sidebar_label: Setup & Identifiers
---

This guide will walk you through setting up Veramo on React Native. You should have a good understanding of React Native and have your environment set up correctly to build iOS and Android apps. Check out the [React Native](https://reactnative.dev/docs/environment-setup) docs to learn more.

## Introduction

Let's setup Veramo to run locally on the device and use `sqlite` to store data, identities, and keys. Our identity provider will be `ethr-did`. Initially, we will set up the [agent](/docs/veramo_agent/introduction) in the most basic config and add more plugins for additional functionality as we go. Right now we just want to create an [identifer](/docs/basics/identifiers).

## Bootstrap React Native

Use the React Native CLI bootstrap a new typescript react project:

```bash
npx react-native init VeramoMobile --template react-native-template-typescript
```

Ensure your project is building and running ok before continuing to next step.

## Install Dependencies

### Native

We need to set up some native dependencies and shims that Veramo plugins will use for database and key management.

```bash
yarn add react-native-sodium react-native-sqlite-storage
```

To access node methods we need to install [rn-nodeify](https://www.npmjs.com/package/rn-nodeify) to our dev dependencies.

```bash
yarn add rn-nodeify --dev
```

Add the following snippets to your package.json file

```json
{
    "scripts": {
    ...
        "postinstall": "rn-nodeify --install assert,buffer,process,crypto,stream,vm --hack"
    }
}
```

Install a TextEncoder polyfill and this will also trigger the `postinstall` script to install additional dependencies based on the `rn-nodeify` command to your app.

```bash
yarn add @zxing/text-encoding
```

Rename `index.js` to `index.ts`.

Import `shim.js` (created by rn-nodify) and `@zxing/text-encoding` into the top of `index.ts`. Please refer to this [issue regarding the TextEncoder polyfill](https://github.com/uport-project/veramo-website/issues/33).

```ts
import './shim'
import '@zxing/text-encoding'
...
```

Open `shim.js` and uncomment `require('crypto)`

```js
// If using the crypto shim, uncomment the following line to ensure
// crypto is loaded first, so it can populate global.crypto
require('crypto')
```

Install all of the pods in your project that came with the new dependencies.

```bash
npx pod-install
```

### Veramo

Now let's install Veramo Core and some plugins. Don't worry; we will walk through what each of these plugins does in the next section.

```bash
yarn add @veramo/core @veramo/did-manager @veramo/kms-local-react-native @veramo/did-provider-ethr @veramo/key-manager @veramo/did-resolver @veramo/data-store @veramo/credential-w3c ethr-did-resolver web-did-resolver
```

:::warning
The latest version of `TypeOrm` breaks in React Native. To get around this you need to install the lastest working version `v0.2.24`. See more about this [issue here](https://github.com/uport-project/veramo/issues/373)
:::

```bash
yarn add typeorm@0.2.24
```

Add a `resolutions` key to your `package.json` file:

```json
"resolutions:" {
  "typeorm": "0.2.24"
}

```

Close the react native packager, clean the project, and rerun your app. If everything is okay, you should see the default React Native screen as before.

## Bootstrap Veramo

We bootstrap Veramo by creating a setup file and initializing the agent. Create a setup file in `src/veramo/setup.ts` and import the following dependencies:

```tsx
// Core interfaces
import { createAgent, IDIDManager, IResolver, IDataStore, IKeyManager } from '@veramo/core'

// Core identity manager plugin
import { DIDManager } from '@veramo/did-manager'

// Ethr did identity provider
import { EthrDIDProvider } from '@veramo/did-provider-ethr'

// Core key manager plugin
import { KeyManager } from '@veramo/key-manager'

// Custom key management system for RN
import { KeyManagementSystem } from '@veramo/kms-local-react-native'

// Custom resolver
// Custom resolvers
import { DIDResolverPlugin } from '@veramo/did-resolver'
import { Resolver } from 'did-resolver'
import { getResolver as ethrDidResolver } from 'ethr-did-resolver'
import { getResolver as webDidResolver } from 'web-did-resolver'

// Storage plugin using TypeOrm
import { Entities, KeyStore, DIDStore, IDataStoreORM } from '@veramo/data-store'

// TypeORM is installed with @veramo/typeorm
import { createConnection } from 'typeorm'

// SQLite setup
import { Initial1616938713828 } from '../migration/1616938713828-initial'
```

Create an infura variable:

```ts
// You will need to get a project ID from infura https://www.infura.io
const INFURA_PROJECT_ID = 'INFURA_PROJECT_ID'
```

Next initilize our sqlite database using TypeORM:

```tsx
// Create react native db connection
const dbConnection = createConnection({
  type: 'react-native',
  database: 'veramo.sqlite',
  location: 'default',
  migrations: [ Initial1616938713828 ],
  migrationsRun: true,
  logging: ['error', 'info', 'warn'],
  entities: Entities,
})
```

Finally, create the agent and add plugins for Key, Identity, Storage, and Resolver.

```tsx
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
          gas: 1000001,
          ttl: 60 * 60 * 24 * 30 * 12 + 1,
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

Awesome! That's the basic agent configured and ready to use. Let's try it out :rocket:

## Basic User Interface

Now that the agent has been created and configured with plugins, we can create some identifiers. For this, we will need some basic UI.

:::note
Veramo does not impose decisions on how you manage state in your app and will work alongside any existing architecture like Redux or Mobx etc. For brevity, we use `useState` in this example, but you can treat Veramo like you would any async data source.
:::

Open `App.tsx` and delete all the contents and add the following code:

```tsx
import React, { useEffect, useState } from 'react'
import { SafeAreaView, ScrollView, View, Text, Button } from 'react-native'

// Import agent from setup
import { agent } from './src/veramo/setup'

interface Identifier {
  did: string
}

const App = () => {
  const [identifiers, setIdentifiers] = useState<Identifier[]>([])

  // Add the new identifier to state
  const createIdentifier = async () => {
    const _id = await agent.didManagerCreate()
    setIdentifiers((s) => s.concat([_id]))
  }

  // Check for existing identifers on load and set them to state
  useEffect(() => {
    const getIdentifiers = async () => {
      const _ids = await agent.didManagerFind()
      setIdentifiers(_ids)

      // Inspect the id object in your debug tool
      console.log('_ids:', _ids)
    }

    getIdentifiers()
  }, [])

  return (
    <SafeAreaView>
      <ScrollView>
        <View style={{ padding: 20 }}>
          <Text style={{ fontSize: 30, fontWeight: 'bold' }}>Identifiers</Text>
          <View style={{ marginBottom: 50, marginTop: 20 }}>
            {identifiers && identifiers.length > 0 ? (
              identifiers.map((id: Identifier) => (
                <View key={id.did}>
                  <Text>{id.did}</Text>
                </View>
              ))
            ) : (
              <Text>No identifiers created yet</Text>
            )}
          </View>
          <Button onPress={() => createIdentifier()} title={'Create Identifier'} />
        </View>
      </ScrollView>
    </SafeAreaView>
  )
}

export default App
```

Close the packager and rebuild the app. Once loaded hit the **Create identifier** button a few times and you should see your identifiers being created!
