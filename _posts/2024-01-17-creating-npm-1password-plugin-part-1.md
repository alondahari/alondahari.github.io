---
title: Creating an NPM 1Password shell plugin
image: img/1password-plugins.png
categories:
  - Open Source
tags:
  - Go
  - CLI
  - 1Password
---

I love 1Password. I especially love their CLI integration with [shell plugins](https://developer.1password.com/docs/cli/shell-plugins). Recently, I decided to dedicate some of my time to contribute to open source projects that I find interesting, so I naturally started looking at the tools I use daily.

## tl;dr

Take a look at the [open PR](https://github.com/1Password/shell-plugins/pull/422) on the GitHub repo for the end result.

## Exploration

Looking through their [open source shell plugins repo](https://github.com/1Password/shell-plugins) on GitHub, I quickly found that there is no plugin for NPM. There was a [previous attempt](https://github.com/1Password/shell-plugins/pull/168) for it, but that was deserted. However, I could see that the internal team was excited about the plugin from that open PR, so I got to working.

The plugin system is implemented using Go, which I only know from minor usage in the past. However, the [tutorial](https://developer.1password.com/docs/cli/shell-plugins/contribute) is very good, and the interface is declarative, so it seemed like it wouldn’t be a huge lift.

First, I familiarized myself with NPM’s authentication through their documentation, which is pretty good. From the internal team’s comments, I could see that an initial attempt to just support the official NPM registry would be a good starting point.

## Initial Setup

I started out with a couple of initial goals:

- I want to verify that the plugin works with the most basic use case, which is to install a package from my private account with the secret from 1Password.
- I only care about npmjs.org as a backend.
- I’m not worried about getting the secret from an existing `.npmrc` file for now.
- I’m not worried about specifying exactly all the commands that need auth right now.
- I’m not worried about MFA for now.

After cloning the 1Password repo and running the `make` command for a new plugin, I was presented with a pretty straight forward boilerplate, where I only needed to update some of the fields to test the plugin out.

The first file is the plugin definition, which includes basic information about the plugin and the platform it represents. After updating the fields I was left with this:

```golang
package npm

import (
  "github.com/1Password/shell-plugins/sdk"
  "github.com/1Password/shell-plugins/sdk/schema"
)

func New() schema.Plugin {
  return schema.Plugin{
    Name: "npm",
    Platform: schema.PlatformInfo{
      Name:     "NPM",
      Homepage: sdk.URL("https://npmjs.com"),
    },
    Credentials: []schema.CredentialType{
      AccessToken(),
    },
    Executables: []schema.Executable{
      NPMCLI(),
    },
  }
}
```

As you can see, pretty straight forward. The one thing to note is that the credential type is `AccessToken`, which is what you get from NPM in your `.npmrc` after you run `npm login`.

The second file is the credentials' definition file, which in my case is a definition for the access token. This file specifies how the credentials will be stored in 1Password as a secret, and how those will be used with the plugin:

```golang
package npm

import (
  "context"

  "github.com/1Password/shell-plugins/sdk"
  "github.com/1Password/shell-plugins/sdk/importer"
  "github.com/1Password/shell-plugins/sdk/provision"
  "github.com/1Password/shell-plugins/sdk/schema"
  "github.com/1Password/shell-plugins/sdk/schema/credname"
  "github.com/1Password/shell-plugins/sdk/schema/fieldname"
)

func AccessToken() schema.CredentialType {
  return schema.CredentialType{
    Name:          credname.AccessToken,
    DocsURL:       sdk.URL("https://docs.npmjs.com/creating-and-viewing-access-tokens"),
    ManagementURL: sdk.URL("https://www.npmjs.com/settings/<username>/tokens/"),
    Fields: []schema.CredentialField{
      {
        Name:                fieldname.Token,
        MarkdownDescription: "Token used to authenticate to NPM.",
        Secret:              true,
        Composition: &schema.ValueComposition{
          Length: 36,
          Prefix: "npm_",
          Charset: schema.Charset{
            Uppercase: true,
            Lowercase: true,
            Digits:    true,
          },
        },
      },
    },
    DefaultProvisioner: provision.EnvVars(defaultEnvVarMapping)
}

var defaultEnvVarMapping = map[string]sdk.FieldName{
  "NPM_CONFIG_//registry.npmjs.org/:_authToken": fieldname.Token,
}
```

We can see here that the 1Password secret will only have one field, which is `token`.
There is also a description of how that token will look like, which I filled in after looking at a generated token from NPM.
The provisioner describes that the credential will be set as an environment variable when running the NPM commands, which is preferable over a temporary file since it’s more ephemeral.
The name for the environment variable might seem a bit strange, [see here](https://github.com/npm/cli/issues/3985#issuecomment-1195946239) for more details on that.

The last file is the executable definition, which looks like this:

```golang
package npm

import (
  "github.com/1Password/shell-plugins/sdk"
  "github.com/1Password/shell-plugins/sdk/schema"
  "github.com/1Password/shell-plugins/sdk/schema/credname"
)

func NPMCLI() schema.Executable {
  return schema.Executable{
    Name:    "NPM CLI",
    Runs:    []string{"npm"},
    DocsURL: sdk.URL("https://docs.npmjs.com/cli"),
    Uses: []schema.CredentialUsage{
      {
        Name: credname.AccessToken,
      },
    },
  }
}
```

Pretty similar to the first file. In the future we will add here the `NeedsAuth` parameter which specifies which commands from the CLI need auth, but we won’t worry about it for now.

There is also a test file, but I’ll get to that later.

## Manual testing

To test the plugin, I first ran `make npm/validate` to make sure everything was in order, and then `make npm/build` to build it locally. This command built the plugin in my home configs, usually at `~/.op/plugins/local` on a Mac.

To initiate the plugin locally, I ran `op plugin init npm` to create the secret on my 1password account, and used an access token I created on my NPM account. Make sure to remove any local `.npmrc` files, so they are not used.
After that, it was simply a matter of running `npm whoami` to verify my token was used successfully. It worked!

_<script async id="asciicast-633115" src="https://asciinema.org/a/633115.js"></script>_

This concludes the first post of this series, I will add another post for each improvement I make for this plugin. I plan those to be:

- [Specifying which NPM commands need auth]({% link _posts/2024-01-24-npm-1password-plugin-part-2.md %})
- [Parsing the secret from an existing `.npmrc` file and supporting different backends]({% link _posts/2024-01-24-npm-1password-plugin-part-3.md %}).
- Handling MFA.
