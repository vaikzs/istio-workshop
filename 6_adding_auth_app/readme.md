## Part 6: Adding authentication to our app

In this part of the guide we will be using Istio to add authentication to our app by verifying Json Web Tokens (JWT). This part will no go in to detail on how to authenticate a user by e-mail and password, and how to sign tokens. We will touch on using Json Web Keys (JWK) with JWT and Istio.  
At the end of this part you will have an application that serves public keys in JWK format, and an endpoint which is protected by Istio and only accessible with the right JWT.

### Asymmetric keys

Most JWT examples use a shared key to both sign and verify the token. This often works fine, however when someone else is required to verify a token. You must either share the key or create an endpoint for it. With Asymmetric keys, this is not the case. Asymmetric keys, such as RSA, comes with a public key and a private key. The private key will be used to sign tokens and must not be shared, but the public key can only be used to verify tokens. With this mechanic anyone can verify if a token is signed by the issuer.

Pretty much all big companies have there public keys up for grabbing. Want to see Google's? [Here it is](https://www.googleapis.com/oauth2/v3/certs). In fact, those are several public keys in JWK format. Now whenever someone comes to us with a JWT saying it is signed by google, we can actually verify that it is, but we can not modify it! If this sounds familiar, that is because OAuth works with the same concept.

### Creating keys and JWK

Creating asymmetric keys is not difficult, but you require some tools. We generate a private key in the PEM format from which you can extract the public key. Most linux systems come with OpenSSL installed. You can generate your keys with the following command. Make sure you do not provide a password.

```
$ openssl genrsa -out private.pem 1024
```

On windows you can use Putty to generate RSA keys. On the following download page, search for `puttygen.exe` and download it. ([Download here](https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html)) Once you open puttygen you should be shown a simple gui, make sure to not provide a password and that you have RSA selected. Then press generate and save the private key.

**Optionally** you can save the public key as well, on linux run `openssl rsa -in private.pem -pubout -out public.pem` and on windows in puttygen click "save
public key".

We will be generating the JWK set (JWKS) at runtime using Cisco's Node-JOSE library. JOSE is an abbreviation for **J**avascript **O**bject **S**igning and **E**ncryption.

With Node-JOSE we will create a keystore which holds the currently active keys. Once we add our private key, the library will do the conversion to JWK and Public Key for you.

```js
// Import FileSystem with promises
const { promises: fs } = require("fs");
// Import JWK from node-jose
const { JWK } = require("node-jose");

// Create a keystore
const keystore = JWK.createKeyStore();

// Load our key
async function loadKeys() {
  // Load the key from the file
  const privateKeyPEM = await fs.readFile("./private.pem");
  // Add the key to the keystore,
  // specifying that it is in PEM format
  await keystore.add(privateKeyPEM, "pem");
}

// Load the keys, and after that print the JWKS
loadKeys().then(() => {
  // By default toJSON will only output the public keys in JWK format
  console.log(keystore.toJSON());
});
```

### Adding a JWKS endpoint to our app

As we have seen, loading keys is not difficult. I have rewritten the above code into [`keyProvider.js`](./src/keyprovider.js) which exposes two functions: `loadKeyFile` and `getJWKS`.

At first I load the keys before starting the GRPC or HTTP server:

```js
const keyProvider = require("./keyprovider");
...
async function main() {
  await keyProvider.loadKeyFile("./private.pem");
  console.log("Loaded keys!");
  ...
}
```

And in the `httpserver.js` I added the endpoint `/.well-known/jwks.json` which calls keyProvider.getJWKS.

```js
const keyProvider = require("./keyprovider");
...
function addRoutes(server) {
...
  // Register the `/.well-known/jwks.json` endpoint
  server.get("/.well-known/jwks.json", (req, res) => {
    const jwks = keyProvider.getJWKS();
    res.send(jwks);
  });
...
}
```

Now using [httpie](https://httpie.org/) I can send a GET request to the endpoint and we should receive the public key:

```json
$ http get :3000/.well-known/jwks.json
...
{
    "keys": [
        {
            "e": "AQAB",
            "kid": "5hi5m4OL-aF_m43tW_j5TOIkUyWXyghpsc24Wq1L11M",
            "kty": "RSA",
            "n": "2BMRH3w8T64Bxqvvh_W7sLhnU6labdvKpEKQ_--qtZnCB7LUsITH2rqU6zVZhf9Y-FoS4zltbiUAczm8MFVWtXKL8yKjWxyW7ylK4mGZE9-Iu8uSmWvsLUoSkSlsDj8T3qghJbf2LmMJpy2rq0uj5_9ntyL3s5reKGusDSMJWeA2tuVlsQV3hL6V1-HP88HAW-KixoUX6FvYLBM2Gk5h-KprIVVGYKnuZvSvVFYxh4
WyNC5YNUGVv71mFIcG2YEJKUK7E0fPoxO0KN3lNWiXLFuRWcSU7JsogB_rHdDGcYDVUSCATHhnyD17_lwSb8lRjuO82xT1C_k7YIw_1H9kDQ"
        }
    ]
}
```