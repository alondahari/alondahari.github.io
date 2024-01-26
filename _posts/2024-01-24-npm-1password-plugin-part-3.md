---
title: NPM 1Password Plugin - Parsing existing config
image: img/1password-plugins.png
categories:
  - Open Source
---

After creating the basic flow of creating a secret token from scratch, we tackle the part of parsing an existing `.npmrc` file.

_This is part 3 of a series of articles about adding an NPM shell plugin to 1Password.
For more context, please check out [part 1]({% link _posts/2024-01-17-creating-npm-1password-plugin-part-1.md %}) and [part 2]({% link _posts/2024-01-24-npm-1password-plugin-part-3.md %})._

This part is a bit more involved and required me to delve a little into coding in Go.

## Initial implementation

To get going, I wanted the plugin to parse a `.npmrc` file located in the home folder with one line - the token used in npmjs.org registry. This file is the product of running `npm login` on a Mac. Our config file will look something like this:

```txt
//registry.npmjs.org/:_authToken=npm_5G4Rq0YH8ptkECnO4FZaQatgAtRubd1mgnWK
```

The relevant file is `plugins/npm/access_token.go`, where we will add a function to import the config file from disk and try getting the token from it. Initial implementation looks like this:

```golang
func TryNPMConfigFile() sdk.Importer {
  return importer.TryFile("~/.npmrc", func(ctx context.Context, contents importer.FileContents, in sdk.ImportInput, out *sdk.ImportAttempt) {
    // don't use colon as a delimiter, since it is used in the .npmrc file as a delimiter
    // between the scope, registry and configuration key
    configs, err := ini.LoadSources(ini.LoadOptions{KeyValueDelimiters: "="}, []byte(contents))
    if err != nil {
      out.AddError(err)
    }

    // sections are not supported in .npmrc
    section, err := configs.GetSection(ini.DefaultSection)
    if err != nil {
      out.AddError(err)
    }
    for _, key := range section.Keys() {
      if strings.Contains(key.Name(), "_authToken") {
        out.AddCandidate(sdk.ImportCandidate{
          Fields: map[sdk.FieldName]string{
            fieldname.Token: key.Value(),
          },
        })
      }
    }
  })
}
```

Although the 1password package has a nice utility for parsing INI files (`contents.toINI()`), we need to interact with the [INI package](https://github.com/go-ini/ini) directly.
Since the `.npmrc` file uses colons to delimit between the scope, registry and config key, and colons can be used as a key/value delimiter according to the spec.

If we find a match, we add it as a candidate for 1password to suggest to the user. After adding the importer function to the schema, our schema looks like this:

```golang
func AccessToken() schema.CredentialType {
  return schema.CredentialType{
    Name:          credname.AccessToken,
    ...
    Importer: importer.TryAll(
      TryNPMConfigFile(),
    ),
  }
}
```

After rebuilding the package and retrying in my test project, I'm getting the correct prompting from the plugin:

_<script async id="asciicast-633073" src="https://asciinema.org/a/633073.js"></script>_

## Per-Project Config Support

Since we can have a config file per project, and not just in the home directory, let's support that.
We will refactor the importer function to accept the file path and add two functions, one for getting the project file and one for getting it from the home folder:

```golang
func TryGlobalNPMConfigFile(env string, defaultPath string) sdk.Importer {
  path := os.Getenv(env)
  if path == "" {
    path = defaultPath
  }
  return TryNPMConfigFile(path)
}


func TryNPMConfigFile(path string) sdk.Importer {
  return importer.TryFile(filepath.Join(path, ".npmrc"), func(ctx context.Context, contents importer.FileContents, in sdk.ImportInput, out *sdk.ImportAttempt) {
    ...
  })
}
```

We will then add those to the schema:

```golang
    Importer: importer.TryAll(
      TryNPMConfigFile(""),
      TryGlobalNPMConfigFile("NPM_CONFIG_USERCONFIG", "~"),
    ),
```

Trying that out, with both files present:

_<script async id="asciicast-633257" src="https://asciinema.org/a/633257.js"></script>_

## Support different backends

Until now, we were just supporting the official NPM registry as a backend, but we can do better. Since we have the registry in the key of the config file, we can use that to support different backends, and scopes.

In order for us to do that, we will need to pivot from using env var to provision the secret, and [provision a temp file](https://developer.1password.com/docs/cli/shell-plugins/contribute/#provisioner) instead.

First, we need to add the host and organization (registry and scope) fields to our secret. Those will be optional:

First, we need to add the host and organization (registry and scope) fields to our secret. Those will be optional:

```golang
{
  return schema.CredentialType{
    Name:          credname.AccessToken,
  ...
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
      {
        Name:                fieldname.Organization,
        MarkdownDescription: "The organization the access token is scoped for.",
        Optional:            true,
      },
      {
        Name:                fieldname.Host,
        MarkdownDescription: "The registry host for the npm packages.",
        Optional:            true,
      },
    },
    ...
  }
}

```

We will then add those to the secret when we parse the file:

```golang
  return importer.TryFile(filepath.Join(path, ".npmrc"), func(ctx context.Context, contents importer.FileContents, in sdk.ImportInput, out *sdk.ImportAttempt) {
    ...
    for _, key := range section.Keys() {
      if strings.Contains(key.Name(), "_authToken") {

        keyParts := strings.Split(key.Name(), ":")

        registry := ""
        scope := ""
        if len(keyParts) == 2 {
          registry = strings.Trim(keyParts[0], "/")
        } else if len(keyParts) == 3 {
          registry = strings.Trim(keyParts[1], "/")
          scope = strings.Trim(keyParts[0], "@")
        }

        out.AddCandidate(sdk.ImportCandidate{
          Fields: map[sdk.FieldName]string{
            fieldname.Token:        key.Value(),
            fieldname.Host:         registry,
            fieldname.Organization: scope,
          },
          NameHint: importer.SanitizeNameHint(registry),
        })
      }
    }
  })

```

Notice we also added a name hint above, so the user can have a better idea what they're choosing in case they have multiple files / credentials in the file.

After that, we need to define how the temp file will be constructed:

```golang
func configFile(in sdk.ProvisionInput) ([]byte, error) {
  contents := ""

  if org, ok := in.ItemFields[fieldname.Organization]; ok && org != "" {
    contents += "@" + strings.Trim(org, "@") + ":"
  }
  if host, ok := in.ItemFields[fieldname.Host]; ok && host != "" {
    contents += "//" + strings.Trim(host, "/") + "/:"
  }

  contents += "_authToken="

  if token, ok := in.ItemFields[fieldname.Token]; ok {
    contents += token
  }

  return []byte(contents), nil
}
```

Finally, we swap out the provisioner in our schema:

```golang
    DefaultProvisioner: provision.TempFile(configFile,
      provision.Filename(".npmrc"),
      provision.AddArgs(
        "--userconfig", "{{ .Path }}",
      ),
    ),
```

That's it for now. Not sure if I'm going to tackle MFA, but we'll see. I think I'll try to submit this PR and see what feedback I get from the maintainers, stay tuned!
