---
id: react_native
title: React Native
sidebar_label: React Native
---

This guide will walk you through setting up Veramo on React Native. You should have a good understanding of React Native and have your environment set up correctly to build iOS and Android apps. Check out the [React Native](https://reactnative.dev/docs/environment-setup) docs to learn more.

## Introduction

Let's setup Veramo to run locally on the device and use `sqlite` to store data, identities, and keys. Our identity provider will be `ethr-did`. Initially, we will set up the [agent](/docs/agent/introduction) in the most basic config and add more plugins for additional functionality as we go. Right now we just want to create an [identifer](/docs/fundamentals/identifiers).

## Bootstrap React Native

Use the React Native CLI bootstrap a new typescript react project:

```bash
npx react-native init Veramo --template react-native-template-typescript
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

Run yarn again to trigger the `postinstall` script. This will install some additional dependencies based on the `rn-nodeify` command to your app.

```bash
yarn
```

Install all of the pods in your project that came with the new dependencies.

```bash
npx pod-install
```

Close the react native packager, clean the project, and rerun your app. If everything is okay, you should see the default React Native screen as before.

### Veramo

Now let's install Veramo Core and some plugins. Don't worry; we will walk through what each of these plugins does in the next step.

```bash
yarn add @veramo/core @veramo/credential-w3c @veramo/data-store @veramo/did-manager @veramo/did-provider-ethr @veramo/did-provider-web @veramo/did-resolver @veramo/key-manager @veramo/kms-local ethr-did-resolver web-did-resolver
```

## Bootstrap Veramo

We bootstrap Veramo by creating a setup file and initializing the agent. Create a setup file in `src/veramo/setup.ts` and import the following dependencies:

```tsx
// Core interfaces
import { createAgent, IDIDManager, IResolver, IDataStore, IKeyManager } from '@veramo/core'

// Core identity manager plugin
import { DIDManager } from '@veramo/did-manager'

// Ethr did identity provider
import { EthrDIDProvider } from '@veramo/plugin-ethr-did'

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

// TypeORM is installed with daf/typeorm
import { createConnection } from 'typeorm'
```

Next initilize our sqlite database using TypeORM:

```tsx
// Create react native db connection
const dbConnection = createConnection({
  type: 'react-native',
  database: 'veramo.sqlite',
  location: 'default',
  synchronize: true,
  logging: ['error', 'info', 'warn'],
  entities: Entities,
})
```

Finally, create the agent and add plugins for Key, Identity, Storage, and Resolver. You will need to get an infura projectId from [Infura](https://infura.io/)

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
          rpcUrl: 'https://rinkeby.infura.io/v3/' + infuraProjectId,
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
      console.log(ids)
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
```

Close the packager and rebuild the app. Once loaded hit the `Create identifier` button a few times and you should see your identifiers being created!

## Verifiable Credentials

So now we can create identifiers, store them in a database, and query for them. Next, we will use an identifier to create some Verifiable Credentials, save them to the database, and query for them too.
