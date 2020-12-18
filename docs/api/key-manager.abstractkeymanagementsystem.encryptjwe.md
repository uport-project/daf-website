---
id: key-manager.abstractkeymanagementsystem.encryptjwe
title: AbstractKeyManagementSystem.encryptJWE() method
hide_title: true
---

<!-- Do not edit this file. It is automatically generated by API Documenter. -->

## AbstractKeyManagementSystem.encryptJWE() method

<b>Signature:</b>

```typescript
abstract encryptJWE(args: {
        key: IKey;
        to: Omit<IKey, 'kms'>;
        data: string;
    }): Promise<string>;
```

## Parameters

| Parameter | Type                                                                                                   | Description |
| --------- | ------------------------------------------------------------------------------------------------------ | ----------- |
| args      | { key: [IKey](./core.ikey.md) ; to: Omit&lt;[IKey](./core.ikey.md)<!-- -->, 'kms'&gt;; data: string; } |             |

<b>Returns:</b>

Promise&lt;string&gt;