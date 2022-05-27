# `__THIS IS A FORK FROM renarsvilnis/apple-sign-in-rest. ONLY CHANGE I HAVE MADE IS ADDING id_token AND refresh_token TO RefreshTokenResponse. THIS VERSION CAN BE DOWNLOADED FROM npm AS apple-sign-in-module__`

# apple-sign-in-module

[![NPM](https://nodei.co/npm/apple-sign-in-module.png?downloads=true&downloadRank=true&stars=true)](https://nodei.co/npm/apple-sign-in-module/)

Supports Node.js `>= 10.x.x`

## Installation

Install the module using [npm](http://npmjs.com):

```bash
npm install --save apple-sign-in-module
yarn add apple-sign-in-module
```

## Documentation

Library is built on typescript and has well documented source code. This will provide a zero-effort developer expierence within your existing code editors. But the library also provides autogenered documentation using [typedoc](https://typedoc.org/).

> [Documentation](https://renarsvilnis.github.io/apple-sign-in-rest/)

## Usage

### 0. Prerequisites

Here are some usefull links to get started with Sign In with Apple:

- [What the Heck is Sign In with Apple?](https://developer.okta.com/blog/2019/06/04/what-the-heck-is-sign-in-with-apple) - good step by step guide, that shows how to create all needed credentials trough Apple Developer Website with screenshots

- [Why sign-in with apple may take you more than 5 minutes and how it works?](https://dev.to/michalrogowski/why-sign-in-with-apple-may-take-you-more-than-5-minutes-and-how-it-works-55p6) - guide on how to implement login using iOS app

- [Apple offical docs](https://help.apple.com/developer-account/#/dev1c0e25352)

- [Sign in with Apple Tutorial](https://sarunw.com/posts/sign-in-with-apple-1/) - 4 part blog that goes trough implementing Sign in with Apple trough both iOS and Web. Also shows how to configure your email service to send emails trough appls relay server

###

### 1. Create a `AppleSignIn` instance

Compared to other libraries `apple-sign-in-rest` chooses to create an instance with credentials instead of passing same credentials to functions on each call. This allows the library to validate them once and create apple public key cache per instance.

```javascript
// Using modules
import {AppleSignIn} from 'apple-sign-in-rest';
// or if using common.js
const {AppleSignIn} = require("apple-sign-in-rest");

/**
 * See docs for full list of options and descriptions:
 * https://renarsvilnis.github.io/apple-sign-in-rest/classes/_applesignin_.applesignin.html#constructor
 */
const appleSignIn = new AppleSignIn({
  /**
   * The clientId depends on that login "flow" you trying to create:
   *   - "web login" - this is the "serviceId"
   *   - "ios login" - this is the app "bundleId", choose only this if you trying to
   *                   verify user that has signed into using the native iOS way
   *
   */
  clientId: "com.my-company.my-app",
  teamId: "5B645323E8",
  keyIdentifier: "U3B842SVGC",
  privateKey: "-----BEGIN PRIVATE KEY-----\nMIGTAgEHIHMJKJyqGSM32AgEGC...",
  // or instead of privateKey use privateKeyPath to read key from file
  privateKeyPath: '/Users/arnold/my-project/credentials/AuthKey.p8'
})
```

### 2. _(Optional)_ Get authorization URL

Start "Sign in with Apple" flow by redirecting user to the authorization URL.

Alternatively, you can use [Sign In with Apple](https://developer.apple.com/documentation/signinwithapplejs) browser javascript library to handle this.

> If you implement in Sign In With Apple trough iOS native login, you won't need to do this.

```javascript
/**
 * See docs for full list of options and descriptions:
 * https://renarsvilnis.github.io/apple-sign-in-rest/classes/_applesignin_.applesignin.html#getauthorizationurl
 */
const authorizationUrl = appleSignIn.getAuthorizationUrl({
  scope: ["name", "email"],
  redirectUri: "http://localhost:3000/auth/apple/callback",
  // (Optional) Value of the anti-forgery unique session token, as well as any other information needed to recover the context when the user returns to your application, e.g., the starting URL.
  state: "123",
  // (Optional) A random value generated by your app that enables replay protection when present.
  nonce: "insert-generated-uuid",
});
```

### 2. Get access token

#### 2.1. Get credentials return after successful sign in with Apple

Retrieve the "code" query parameter from the URL string when the user user redirect to your sites previously provided `redirectUrl` after a successfull Sign In With Apple.

> Make sure to verify that the `state` matches what was set when creating the authorization url!

```javascript
// Example: http://localhost:3000/auth/apple/callback?code=somecode&state=123
app.get("/auth/apple/callback", (req, res) => {
  const code = req.query.code;
  const state = req.query.state;

  // This depends how you implemented the storing "state"
  if (req.session.state && req.session.state !== state) {
    throw new Error("Missing or invalid state");
  }

  // Continues in next examples...
});
```

In the case of iOS the native-app sends the `code` to a custom endpoint on your app. Make sure that iOS app also securily passes along the `fullName` and `idToken` . From the backend you won't have access to the `fullName` and `idToken` can be used to verify the user without making `appleSignIn.getAuthorization()` call.

#### 2.2. Create a `userSecret`

```javascript
/**
 * See docs for full list of options and descriptions:
 * https://renarsvilnis.github.io/apple-sign-in-rest/classes/_applesignin_.applesignin.html#createclientsecret
 */
const clientSecret = appleSignIn.createClientSecret({
  /**
   * Optionaly you can set the validity duration of the secret in seconds. Apple allows the secret to up to 6 months,
   * but if you are creating a clientSecret per request basis you can set your own expiration duration.
   * Defaults to 6 months.
   */
  expirationDuration: 5 * 60, // 5 minutes
});
```

#### 2.2. Exchange retrieved "code" to user's access token.

```javascript
/**
 * See docs for full list of options and descriptions:
 * https://renarsvilnis.github.io/apple-sign-in-rest/classes/_applesignin_.applesignin.html#getauthorizationtoken
 */
const tokenResponse = await appleSignIn.getAuthorizationToken(clientSecret, code, {
  // Optional, use the same value which you passed to authorisation URL. In case of iOS you skip the value
  redirectUri: "http://localhost:3000/auth/apple/callback",
});
```

Result of `appleSignIn.getAuthorizationToken` command is a JSON object representing Apple's [TokenResponse](https://developer.apple.com/documentation/signinwithapplerestapi/tokenresponse):

```javascript
{
   // A token used to access allowed data. Currently has no use
    access_token: "ACCESS_TOKEN",
    // It will always be Bearer.
    token_type: 'Bearer',
    // The amount of time, in seconds, before the access token expires.
    expires_in: 3600,
    // used to regenerate new access tokens. Store this token securely on your server.
    refresh_token: "REFRESH_TOKEN",
    // A JSON Web Token that contains the user’s identity information.
    id_token: "ID_TOKEN"
}
```

#### 2.4. Verify token signature and get unique user's identifier

```javascript
/**
 * See docs for full list of options and descriptions:
 * https://renarsvilnis.github.io/apple-sign-in-rest/classes/_applesignin_.applesignin.html#verifyidtoken
 */
const claim = await appleSignIn.verifyIdToken(tokenResponse.id_token, {
  // (Optional) verifies the nonce
  nonce: "nonce",
  // (Optional) If you want to check subject(sub field) of the jwt a.k.a "user_identifier|provider_id". Might be usefull in the case where iOS app also sends you it, so you can verify if it is correct
  subject: "000852.5g3d8d4b3db045b48b7a58fb07728e1e.1303",
  // (Optional) If you want to handle expiration on your own, useful in case of iOS as identityId can't be "refreshed"
  ignoreExpiration: true, // default is false
});
```

### 3. Refresh access token after expiration

> Should call it no more once a day, else apple amy throttle your request.

```javascript
/**
 * See docs for full list of options and descriptions:
 * https://renarsvilnis.github.io/apple-sign-in-rest/classes/_applesignin_.applesignin.html#refreshauthorizationtoken
 */
const refreshTokenResponse = await appleSignIn.refreshAuthorizationToken(clientSecret, refreshToken);
const { access_token } = refreshTokenResponse.access_token;
```

## Comparison to other "apple sign in" libraries

There are many already packages on npm with very similar names. Most of them are missing features and/or abandoned. This package takes inspiration from `apple-signin` and implements features/fixes while comparing to other libraries.

The only other library I'd consider feature-full and ready to use besides this one is [apple-signin-auth](https://github.com/A-Tokyo/apple-signin-auth) by [A-Tokyo](https://github.com/A-Tokyo), seem to have missing key features.

|                               | apple-sign-in-rest                                                                                                                          | [apple-signin-auth](https://github.com/A-Tokyo/apple-signin-auth)                                                                         | [apple-auth](https://github.com/ananay/apple-auth)                                                                          | [apple-signin](https://github.com/Techofficer/node-apple-signin)                    |
| ----------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------- |
| Feature Full                  | ✅                                                                                                                                          | ✅ (missing some minor options)                                                                                                           | ❌                                                                                                                          | ❌                                                                                  |
| Apple Public Key Caching      | ✅ (cache per class instance)                                                                                                               | ✅ (global cache)                                                                                                                         | ❌                                                                                                                          | ❌                                                                                  |
| Passport.js library           | ❌ (comming-soon)                                                                                                                           | ❌                                                                                                                                        | ✅                                                                                                                          | ✅                                                                                  |
| Typed Support                 | ✅ (typescript based)                                                                                                                       | ✅ (flow based)                                                                                                                           | ❌                                                                                                                          | ❌                                                                                  |
| API Documentation             | ✅ (auto generated docs using [typedoc](https://typedoc.org/))                                                                              | ❌                                                                                                                                        | ❌                                                                                                                          | ❌                                                                                  |
| Usage Examples                | ✅                                                                                                                                          | ✅                                                                                                                                        | ✅                                                                                                                          | ✅                                                                                  |
| Tools for easier contributors | ✅ (typescript, eslint, prettier, jest)                                                                                                     | ✅ (flow, eslint, prettier, jest)                                                                                                         | ❌                                                                                                                          | ❌                                                                                  |
| Stats                         | [![NPM](https://nodei.co/npm/apple-sign-in-rest.png?downloads=true&downloadRank=true&stars=true)](https://nodei.co/npm/apple-sign-in-rest/) | [![NPM](https://nodei.co/npm/apple-signin-auth.png?downloads=true&downloadRank=true&stars=true)](https://nodei.co/npm/apple-signin-auth/) | [![NPM](https://nodei.co/npm/apple-auth.png?downloads=true&downloadRank=true&stars=true)](https://nodei.co/npm/apple-auth/) | [![NPM](https://nodei.co/npm/apple-signin.png)](https://nodei.co/npm/apple-signin/) |

## Contributing

Pull requests are always welcomed. 🙇🏻‍♂️ Please open an issue first to discuss what you would like to change.

Package has a pre-commit git hook that does typechecking, linting, unit testing and doc building (if see source code changes).

### Helper scripts

```bash
# Build library, will create a library in /lib folder
npm run build

# Run unit tests
npm run test
npm run test:watch # watch mode

# Run typecheck and linter
npm run lint

# Attempts to fix all formatting and linting issues
npm run format

# Build docs
npm run docs

# Inspect documentation localy visit http://127.0.0.1:8080
npm run docs:serve

# By default docs are automatically built and added on pre-commit hook,
# if it sees staged changes to any /src files,
# you can override the logic by forcing to build docs by passing environmental
DOCS_FORCE=true git commit -m 'My awesome change'

# You also skip automatically adding docs to commit
DOCS_COMMIT=false git commit -m 'My awesome change'

# Commit but ignore ship the git hooks
git commit -m 'My awesome change' --no-verify
```

## License

[The MIT License](https://choosealicense.com/licenses/mit/)

Copyright (c) 2020 Renārs Vilnis

## Support

If you have any questions or need help with integration, then you can contact me by email
[renars.vilnis@gmail.com](renars.vilnis@gmail.com).
