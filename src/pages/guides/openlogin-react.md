---
title: How to Add OpenLogin to a React app
image: "/contents/openlogin-react.png"
description: Learn to build a simple React.js app secured by OpenLogin.
order: 1
---

import Tabs from "@theme/Tabs";

import TabItem from "@theme/TabItem";

## Instruction

This tutorial will show you how to integrate OpenLogin authentication into one
of the most popular web front-end library: React.

## Create your React app

> Ignore this step if you're integrating into an existing React project.

<Tabs
  defaultValue="npm"
  values={[
    { label: "npm", value: "npm" },
    { label: "Yarn", value: "yarn" },
  ]}
>
<TabItem value="npm">

```shell
npx create-react-app my-app
cd my-app
npm start
```

</TabItem>

<TabItem value="yarn">

```shell
yarn create react-app my-app
cd my-app
yarn start
```

</TabItem>

</Tabs>

## Install OpenLogin SDK

Install Torus OpenLogin SDK using either npm or yarn:

<Tabs
  defaultValue="npm"
  values={[
    { label: "npm", value: "npm" },
    { label: "Yarn", value: "yarn" },
  ]}
>
<TabItem value="npm">

```shell
npm i --save @toruslabs/openlogin
```

</TabItem>

<TabItem value="yarn">

```shell
yarn add @toruslabs/openlogin
```

</TabItem>

</Tabs>

## Serve service worker

Add this [service worker](https://github.com/torusresearch/torus-direct-web-sdk/blob/master/examples/react-app/public/serviceworker/sw.js) to `/public/serviceworker` directory in your project. This service worker will handle redirect result from OpenLogin website for you. If you're already using a service worker, please port over the fetch override.

If you prefer not using service worker (your customers are using browsers that don't support service worker), you can add this [redirect.html](https://github.com/torusresearch/torus-direct-web-sdk/blob/master/examples/react-app/public/serviceworker/ridirect.html) to `/public` directory to achieve a similar result as using service worker.

## Initialize the SDK on your application's mounted event

Initialize the SDK after your application is mounted using `useEffect` hook and call `openlogin.login` when user clicks login in your application.

```jsx
const VERIFIER = {
  typeOfLogin: "google", // "facebook", "apple", "twitter", "reddit", etc. See full list of supported logins: https://docs.tor.us/direct-auth/supported-authenticators-verifiers
  clientId: "Your social provider ID",
}

function App() {
  const [isLoading, setLoading] = useState(true);

  const [openlogin, setOpenLogin] = useState();
  const [privKey, setPrivKey] = useState();

  const onMount = async () => {
    setLoading(true);

    try {
      const openlogin = new OpenLogin({
        clientId: VERIFIER.clientId,
        iframeUrl: "http://beta.openlogin.com", // Beta version of OpenLogin
      });
      setOpenLogin(openlogin);

      await openlogin.init();
      setPrivKey(openlogin.privKey);
    } finally {
      setLoading(false);
    }
  };

  const onLogin = async () => {
    if (isLoading || privKey) return;

    setLoading(true);
    try {
      await openlogin.login({
        loginProvider: VERIFIER.typeOfLogin,
        redirectUrl: "http://localhost:3000/redirect",
      });
      setPrivKey(openlogin.privKey);
    } finally {
      setLoading(false);
    }
  };

  useEffect(() => {
    onMount();
  }, []);

  if(isLoading) return <div>Loading...</div>;
  return privKey ? (<div>Logged in: {privKey}</div>) : (<button onClick={onLogin}>Login</button>);
}
```

## Use the private key

After calling `openlogin.login` and handling redirect result, your application will have access to the user's private key at `openlogin.privKey`. You can use this private key to sign messages, generate public key to encrypt and decrypt data later, etc.

### Generate public key

You can use [@toruslabs/eccrypto](https://www.npmjs.com/package/@toruslabs/eccrypto) to generate public key from the user's private key:

```js
import eccrypto from "@toruslabs/eccrypto";

// Corresponding uncompressed (65-byte) public key.
const publicKey = eccrypto.getPublic(privateKey);
```

This public key can be shared with everyone who wants to communicate with the user.

### Sign data

The private key can be used to sign data to indicate that a message is sent by the user.

```js
eccrypto.sign(privateKey, msg).then(function(sig) {
  console.log("Signature in DER format:", sig);
});
```

### Verify signature

Everyone with the user's public key can verify if the message is signed by the user. For example, on server side, you can verify a signed message as a way to authenticate the user:

```js
eccrypto.verify(publicKey, msg, sig).then(function() {
  console.log("Signature is OK");
}).catch(function() {
  console.log("Signature is BAD");
});
```