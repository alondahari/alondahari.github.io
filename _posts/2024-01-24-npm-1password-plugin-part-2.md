---
title: NPM 1Password Plugin - Adding needsAuth
image: img/1password-plugins.png
categories:
  - Open Source
---

So we have our basic implementation of the plugin, now it's time to make it better. This next step is pretty short and straight forward. We need to describe in the plugin which commands in our NPM CLI would require auth from 1password.

_This is part 2 of a series of articles about adding an NPM shell plugin to 1Password. For more context, please check out [part 1]({% link _posts/2024-01-17-npm-1password-plugin-part-1.md %})._

An example of when we don't need to authenticate is when the command includes the `--help` or `--version` flags. In our case, when running `npm uninstall` for example, we would never need to authenticate.

The relevant file here is `npm.go`, which now looks like this:

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

We need to add the `NeedsAuth` key, and feed to it utility functions from `sdk/needsauth`. Looking through the source and some examples, I found two functions that seem relevant: `WhenForCommand` and `NotWhenContainsArgs`.

I quickly realized I want to have a blocklist and not an allowlist, since there are a lot of NPM commands and most of them need auth when it comes to dealing with private packages. I also didn't want to have to deal with updating the plugin every time a new command or alias is added to the CLI.

Initially, I thought about maybe adding some utility for `NotWhenForCommand`, since I care more about the different commands vs. the passed arguments. Eventually I decided against it since it seemed like the library creators intentionally didn't want it there, and I didn't want to make any major updates that will add more work for the package maintainers.

Looking at the source code of these functions, I realized that the `NotWhenContainsArgs` function also examines the command part, and not just the arguments, so that should work for our purposes.

After looking through the NPM CLI docs, I came up with this probably incomplete list of commands to skip auth with, which should probably be good enough for most use cases. Worst case, the CLI might end up authenticating when it's not necessary.

```
package npm

import (
  "github.com/1Password/shell-plugins/sdk"
  "github.com/1Password/shell-plugins/sdk/needsauth"
  "github.com/1Password/shell-plugins/sdk/schema"
  "github.com/1Password/shell-plugins/sdk/schema/credname"
)

func NPMCLI() schema.Executable {
  return schema.Executable{
    Name:    "NPM CLI",
    Runs:    []string{"npm"},
    DocsURL: sdk.URL("https://docs.npmjs.com/cli"),
    NeedsAuth: needsauth.IfAll(
      // not a complete list of commands that don't
      // need auth, but probably the main ones
      needsauth.NotForHelpOrVersion(),
      needsauth.NotWhenContainsArgs("init"),
      needsauth.NotWhenContainsArgs("?"),
      needsauth.NotWhenContainsArgs("config"),
      needsauth.NotWhenContainsArgs("help-search"),
      needsauth.NotWhenContainsArgs("login"),
      needsauth.NotWhenContainsArgs("logout"),
      needsauth.NotWhenContainsArgs("prune"),
      needsauth.NotWhenContainsArgs("shrinkwrap"),
      needsauth.NotWhenContainsArgs("start"),
      needsauth.NotWhenContainsArgs("run-script"),
      needsauth.NotWhenContainsArgs("run"),
      needsauth.NotWhenContainsArgs("rum"),
      needsauth.NotWhenContainsArgs("urn"),
      needsauth.NotWhenContainsArgs("uninstall"),
      needsauth.NotWhenContainsArgs("unlink"),
      needsauth.NotWhenContainsArgs("remove"),
      needsauth.NotWhenContainsArgs("rm"),
      needsauth.NotWhenContainsArgs("r"),
      needsauth.NotWhenContainsArgs("un"),
    ),
    Uses: []schema.CredentialUsage{
      {
        Name: credname.AccessToken,
      },
    },
  }
}

```

Stay tuned for the next instalment, where I will tackle parsing an existing `.npmrc` file!
