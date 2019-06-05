# auth-token.js

> Octokit library implementing the token authentication strategy for browsers and Node.js

`@octokit/auth-token` is the simplest of GitHub’s authentication strategies.

A string is passed to the `createTokenAuth` function which returns the async `auth` function.

The `auth` function validates the passed token and resolves with the correct `authorization` header.

## Usage

```js
import { createTokenAuth } from "@octokit/auth-token";
import { request } from "@octokit/request";

(async () => {
  const auth = createTokenAuth("1234567890abcdef1234567890abcdef12345678");
  const authentication = await auth();
  // {
  //   type: 'token',
  //   token: '1234567890abcdef1234567890abcdef12345678',
  //   tokenType: 'oauth',
  //   headers: {
  //     authorization: 'token 1234567890abcdef1234567890abcdef12345678'
  //   }
  // }

  // `authentication.headers` can be directly passed to a request
  const result = await request("GET /orgs/:org/repos", {
    headers: authentication.headers,
    org: "octokit",
    type: "private"
  });
})();
```

## `createTokenAuth(token)`

The `createTokenAuth` method accepts a single argument of type string, which is the token. The passed token can be one of the following:

- [Personal access token](https://help.github.com/en/articles/creating-a-personal-access-token-for-the-command-line)
- [OAuth access token](https://developer.github.com/apps/building-oauth-apps/authorizing-oauth-apps/)
- Installation access token ([GitHub App Installation](https://developer.github.com/apps/building-github-apps/authenticating-with-github-apps/#authenticating-as-an-installation))
- [GITHUB_TOKEN provided to GitHub Actions](https://developer.github.com/actions/creating-github-actions/accessing-the-runtime-environment/#environment-variables)

Examples

```js
// Personal/OAuth access token
createTokenAuth("1234567890abcdef1234567890abcdef12345678");

// Installation access token or GitHub Action token
createTokenAuth("v1.d3d433526f780fbcc3129004e2731b3904ad0b86");
```

It returns the asynchronous `auth()` method.

## `auth()`

The `auth()` method has no options. It returns the authentication object.

## Authentication object

<table width="100%">
  <thead align=left>
    <tr>
      <th width=150>
        name
      </th>
      <th width=70>
        type
      </th>
      <th>
        description
      </th>
    </tr>
  </thead>
  <tbody align=left valign=top>
    <tr>
      <th>
        <code>type</code>
      </th>
      <th>
        <code>string</code>
      </th>
      <td>
        <code>"token"</code>
      </td>
    </tr>
    <tr>
      <th>
        <code>token</code>
      </th>
      <th>
        <code>string</code>
      </th>
      <td>
        The provided token.
      </td>
    </tr>
    <tr>
      <th>
        <code>tokenType</code>
      </th>
      <th>
        <code>string</code>
      </th>
      <td>
        <code>"oauth" for personal access tokens and OAuth tokens, or "installation" for installation access tokens</code>
      </td>
    </tr>
    <tr>
      <th>
        <code>headers</code>
      </th>
      <th>
        <code>object</code>
      </th>
      <td>
        <code>{ authorization } </code> - value for the "Authorization" header.
      </td>
    </tr>
    <tr>
      <th>
        <code>query</code>
      </th>
      <th>
        <code>object</code>
      </th>
      <td>
        <code>{}</code> - not used
      </td>
    </tr>
  </tbody>
</table>

## Find more information

`createTokenAuth` does not send any requests, it only transforms the provided token string into an authentication object.

Here is a list of things you can do to retrieve further information

### Find out what scopes are enabled for oauth tokens

Note that this does not work for installations. There is no way to retrieve permissions based on an installation access tokens.

```js
import { createTokenAuth } from "@octokit/auth-token";
import { request } from "@octokit/request";

const TOKEN = "1234567890abcdef1234567890abcdef12345678";

(async () => {
  const auth = createTokenAuth(TOKEN);
  const authentication = await auth();

  const response = await request("HEAD /", {
    headers: authentication.headers
  });
  const scopes = response.headers["x-oauth-scopes"].split(/,\s+/);

  if (scopes.length) {
    console.log(
      `"${TOKEN}" has ${scopes.length} scopes enabled: ${scopes.join(", ")}`
    );
  } else {
    console.log(`"${TOKEN}" has no scopes enabled`);
  }
})();
```

### Find out if token is a personal access token or if it belongs to an OAuth app

```js
import { createTokenAuth } from "@octokit/auth-token";
import { request } from "@octokit/request";

const TOKEN = "1234567890abcdef1234567890abcdef12345678";

(async () => {
  const auth = createTokenAuth(TOKEN);
  const authentication = await auth();

  const response = await request("HEAD /", {
    headers: authentication.headers
  });
  const clientId = response.headers["x-oauth-client-id"];

  if (clientId) {
    console.log(
      `"${token}" is an OAuth token, its app’s client_id is ${clientId}.`
    );
  } else {
    console.log(`"${token}" is a personal access token`);
  }
})();
```

### Find out what permissions are enabled for a repository

Note that the `permissions` key is not set when authenticated using an installation access token.

```js
import { createTokenAuth } from "@octokit/auth-token";
import { request } from "@octokit/request";

const TOKEN = "1234567890abcdef1234567890abcdef12345678";

(async () => {
  const auth = createTokenAuth(TOKEN);
  const authentication = await auth();

  const response = await request("GET /repos/:owner/:repo", {
    owner: 'octocat',
    repo: 'hello-world'
    headers: authentication.headers
  });

  console.log(response.data.permissions)
  // {
  //   admin: true,
  //   push: true,
  //   pull: true
  // }
})();
```

### Use token for git operations

Both OAuth and installation access tokens can be used for git operations. However when using with an installation, [the token must be prefixed with `x-access-token`](https://developer.github.com/apps/building-github-apps/authenticating-with-github-apps/#http-based-git-access-by-an-installation).

```js
import { createTokenAuth } from "@octokit/auth-token";
import { request } from "execa";

const TOKEN = "1234567890abcdef1234567890abcdef12345678";

(async () => {
  const auth = createTokenAuth(TOKEN);
  const { token, tokenType } = await auth();
  const tokenWithPrefix =
    tokenType === "installation" ? `x-access-token:${token}` : token;

  const repositoryUrl = `https://${tokenWithPrefix}@github.com/octocat/hello-world.git`;

  const { stdout } = await execa("git", ["push", repositoryUrl]);
  console.log(stdout);
})();
```

## License

[MIT](LICENSE)