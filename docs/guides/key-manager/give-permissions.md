---
sidebar_label: 'Grant Permissions to 3rd parties'
sidebar_position: 1
---

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

# Grant Permissions to 3rd party addresses

:::info Requirements

You will need a Universal Profile that you can control via its KeyManager to follow this guide. <br/>
If you don't have a Universal Profile yet, follow our previous guide [**Create a Universal Profile**](../universal-profile/create-profile.md) or look at the [_lsp-factory.js_](../../tools/lsp-factoryjs/deployment/universal-profile.md) docs.

:::

In this guide, we will learn how to grant permissions to third-party addresses to enable them to interact with our Universal Profile. 

By the end of this guide, you will know:

- How permissions in the LSP6 Key Manager work + how to create them using [_erc725.js_](../../../../tools/erc725js/getting-started).
- How to set permissions for a third party `address` on your Universal Profile.

![Give permissions to 3rd parties overview](/img/guides/lsp6/grant-permissions-to-3rd-parties-overview.jpeg)

## Introduction

The Key Manager (KM) enables us to give permissions to other 3rd party addresses to perform certain actions on our Universal Profile (UP), such as editing our profile details, or any other profile metadata.

## Setup

```shell
npm install @erc725/erc725.js @lukso/lsp-smart-contracts
```

## Step 1 - Initialize erc725.js

The first step is to initialize the erc725.js library with a JSON schema specific to the LSP6 Key Manager. This will enable the library to know how to create and encode the permissions.

```js
import { ERC725 } from "@erc725/erc725.js";
import LSP6Schema from "@erc725/erc725.js/schemas/LSP6KeyManager.json";

// step 1 -initialize erc725.js with the ERC725Y permissions data keys from LSP6 Key Manager
const erc725 = new ERC725(LSP6Schema);
```


## Step 2 - Encode the permissions

:::info 

More permissions are available in erc725.js. See the API docs for [`encodePermissions(...)`](../../tools/erc725js/classes/ERC725.md#encodepermissions) function for a complete list.

:::

We can now use erc725.js to create the permissions for a specific 3rd party `address`. The library provide convenience functions for us, such as [`encodePermissions`](../../../../tools/erc725js/classes/ERC725#encodepermissions).

### 2.1 - Create the permission


Let's consider in our example that we want to grant the permission `SETDATA` to a `beneficiaryAddress`, so that it can edit our Universal Profile details on our behalf.

We can do this very easily with erc725.js, using the `encodePermissions(...)` function.

```js
// step 2.1 - create the permissions of the beneficiary address
const beneficiaryPermissions = erc725.encodePermissions({
    SETDATA: true,
});
```

### 2.2 - Encode the permission for the 3rd party address

Now that we have created the permission value `SETDATA`, we need to assign it to the `beneficiaryAddress`.

To do so, we need to assign the permission value created in step 2.1 to the `beneficiaryAddress`, using the `AddressPermissions:Permissions:<address>`, where `<address>` will be the address of the beneficiary

```js
// step 2.2 - encode the data key-value pairs of the permissions to be set for the beneficiary address
const beneficiaryAddress = "0xcafecafecafecafecafecafecafecafecafecafe"; // EOA address of an exemplary person
const permissionData = erc725.encodeData({
    keyName: "AddressPermissions:Permissions:<address>",
    dynamicKeyParts: beneficiaryAddress,
    value: beneficiaryPermissions,
  });
```


## Step 3 - Add the permissions on your UP

We have now all the data needed to setup the permission for this 3rd party addres on our Universal Profile.

### 3.1 - Load your controller address

We will need to interact with the Key Manager from the main controller address (the externally owned account (EOA) that has all the permissions on the UP).

The first step is to load this main controller address as an EOA using its private key.

The private key can be obtained depending on how you created your Universal Profile:

- UP created with our **Create a Universal Profile guide**: you should have provided the private key and known it.
- UP created with the _lsp-factory.js_: this is the private key given in the `controllerAddresses` field in the method [`lspFactory.UniversalProfile.deploy(...)`](../../tools/lsp-factoryjs/classes/universal-profile#deploy)
- UP created via the Browser extension: click on the _Settings_ icon (top right) > and _Export Private Key_


```javascript title="Load account from a private key"
import Web3 from 'web3';
const web3 = new Web3('https://rpc.l16.lukso.network');

const PRIVATE_KEY = '0x...'; // your EOA private key (previously created)

const myEOA = web3.eth.accounts.wallet.add(PRIVATE_KEY);
```

### 3.2 - Create contract instance

The next steps is to create the web3 instance of our smart contracts to interact with them. The contract ABIs are available in the @lukso/lsp-smart-contracts npm package.

You will need the address of your Universal Profile deployed on L16.

```js
import UniversalProfile from "@lukso/lsp-smart-contracts/artifacts/UniversalProfile.json";
import KeyManager from "@lukso/lsp-smart-contracts/artifacts/LSP6KeyManager.json";
import Web3 from "web3";

const RPC_ENDPOINT = "https://rpc.l16.lukso.network";
const web3 = new Web3(RPC_ENDPOINT);


// step 1 - create instance of UniversalProfile and KeyManager contracts
const myUniversalProfileAddress = "0x4F81bFDD12c73c94222C7879C91c1B837b8adb62";
const myUniversalProfile = new web3.eth.Contract(UniversalProfile.abi, myUniversalProfileAddress);
```

Since the KeyManager is the owner of the Universal Profile, its address can be obtained easily by querying the `owner()` of the Universal Profile.

```js
const keyManagerAddress = await myUniversalProfile.methods.owner().call();
const myKeyManager = new web3.eth.Contract(KeyManager.abi, keyManagerAddress);
```


### 3.3 - Set the permission on the Universal Profile

The last and final step is to setup the permissions the `beneficiaryAddress` on our Universal Profile.

We can easily access the data key-value pair from the encoded data obtained by erc725.js in step 2.2.

We will then encode this permission data keys in a `setData(...)` payload and interact via the Key Manager.

```js
// step 3.3 - encode the payload to set permissions and send the transaction via the Key Manager
const payload = myUniversalProfile.methods["setData(bytes32,bytes)"](data.keys[0], data.values[0]).encodeABI();

  // step 4 - send the transaction via the Key Manager contract
  await myKeyManager.methods.execute(payload).send({
    from: myEOA.address,
    gasLimit: 300_000,
  });
```

## Final code

```js
import { ERC725 } from "@erc725/erc725.js";
import LSP6Schema from "@erc725/erc725.js/schemas/LSP6KeyManager.json";
import UniversalProfile from "@lukso/lsp-smart-contracts/artifacts/UniversalProfile.json";
import KeyManager from "@lukso/lsp-smart-contracts/artifacts/LSP6KeyManager.json";
import Web3 from "web3";

// setup
const myUniversalProfileAddress = "0x..."; // the address of your Universal Profile on L16
const RPC_ENDPOINT = "https://rpc.l16.lukso.network";

const web3 = new Web3(RPC_ENDPOINT);
const erc725 = new ERC725(LSP6Schema);

const PRIVATE_KEY = "0x..."; // private key of your main controller address
const myEOA = web3.eth.accounts.wallet.add(PRIVATE_KEY);

async function grantPermissions() {
  // step 1 - create instance of UniversalProfile and KeyManager contracts
  const myUniversalProfile = new web3.eth.Contract(UniversalProfile.abi, myUniversalProfileAddress);

  const keyManagerAddress = await myUniversalProfile.methods.owner().call();
  const myKeyManager = new web3.eth.Contract(KeyManager.abi, keyManagerAddress);
  console.log("keyManagerAddress", keyManagerAddress);

  // step 2 - setup the permissions of the beneficiary address
  const beneficiaryAddress = "0xcafecafecafecafecafecafecafecafecafecafe"; // EOA address of an exemplary person
  const beneficiaryPermissions = erc725.encodePermissions({
    SETDATA: true,
  });

  // step 3.1 - encode the data key-value pairs of the permissions to be set
  const data = erc725.encodeData({
    keyName: "AddressPermissions:Permissions:<address>",
    dynamicKeyParts: beneficiaryAddress,
    value: beneficiaryPermissions,
  });

  console.log(data);

  //   step 3.2 - encode the payload to be sent to the Key Manager contract
  const payload = myUniversalProfile.methods["setData(bytes32,bytes)"](data.keys[0], data.values[0]).encodeABI();

  // step 4 - send the transaction via the Key Manager contract
  await myKeyManager.methods.execute(payload).send({
    from: myEOA.address,
    gasLimit: 300_000,
  });

  const result = await myUniversalProfile.methods["getData(bytes32)"](data.keys[0]).call();
  console.log(
    `The beneficiary address ${beneficiaryAddress} has now the following permissions:`,
    erc725.decodePermissions(result)
  );
}

grantPermissions();
```


## Testing the permissions

We can now check that the permissions have been correctly set by querying the `AddressPermissions:Permissions:<beneficiaryAddress>` data key on the ERC725Y storage of the Universal Profile.

If everything went well, the code snippet below should return you back an object with the permission `SETDATA` set to `true`.

```js
const result = await myUniversalProfile.methods["getData(bytes32)"](data.keys[0]).call();
  console.log(
    `The beneficiary address ${beneficiaryAddress} has now the following permissions:`,
    erc725.decodePermissions(result)
  );
```

Finally, to test the actual permissions, you can do this guide using a `beneficiaryAddress` that you have control over (created manually via web3.js).

You can then try to do again the **Edit our Universal Profile** guide, using this new 3rd party address that you have control over to test if it can successfull edit the profile details. This will give you guarantee that this `beneficiaryAddress` has the `SETDATA` permission working.



