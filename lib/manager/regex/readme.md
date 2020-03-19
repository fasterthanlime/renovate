The `regex` manager is designed to allow users to manually configure Renovate for how to find dependencies that aren't detected by the built-in package managers.

This manager is unique in Renovate in that:

- It is configurable via regex named capture groups
- Through the use of the `regexManagers` config, multiple "regex managers" can be created for the same repository.

### Required Fields

The first two required fields are `fileMatch` and `matchStrings`. `fileMatch` works the same as any manager, while `matchStrings` is a `regexManagers` concept and is used for configuring a regular expression with named capture groups.

In order for Renovate to look up a dependency and decide about updates, it then needs the following information about each dependency:

- The dependency's name
- Which `datasource` to look up (e.g. npm, Docker, GitHub tags, etc)
- Which version scheme to apply (defaults to `semver`, but also may be other values like `pep440`)

Configuration-wise, it works like this:

- You must capture the `currentValue` of the dependency in a named capture group
- You must have either a `depName` capture group or a `depNameTemplate` config field
- You can optionally have a `lookupName` capture group or a `lookupNameTemplate` if it differs from `depName`
- You must have either a `datasource` capture group or a `datasourceTemplate` config field
- You can optionally have a `versioning` capture group or a `versioningTemplate` config field. If neither are present, `semver` will be used as the default

### Regular Expression Capture Groups

To be fully effective with the regex manager, you will need to understand regular expressions and named capture groups, although sometimes enough examples can compensate for lack of experience.

Consider this `Dockerfile`:

```
FROM node:12
ENV YARN_VERSION=1.19.1
RUN curl -o- -L https://yarnpkg.com/install.sh | bash -s -- --version ${YARN_VERSION}
```

You would need to capture the `currentValue` using a named capture group, like so: `ENV YARN_VERSION=(?<currentValue>.*?)\n`.

If you're looking for an online regex testing tool that supports capture groups, try [https://regex101.com/](https://regex101.com/).

### Configuration templates

In many cases, named capture groups alone won't be enough and you'll need to configure Renovate with additional information about how to look up a dependency. Continuing the above example with Yarn, here is the full config:

```json
{
  "regexManagers": [
    {
      "fileMatch": ["^Dockerfile$"],
      "matchStrings": ["ENV YARN_VERSION=(?<currentValue>.*?)\n"],
      "depNameTemplate": "yarn",
      "datasourceTemplate": "npm"
    }
  ]
}
```

### Advanced Capture

Let's say that your `Dockerfile` has many `ENV` variables you want to keep updated and you prefer not to write one `regexManagers` rule per variable. Instead you could enhance your `Dockerfile` like the following:

```
ENV NODE_VERSION=10.19.0 # github-tags/nodejs/node&versioning=node
ENV COMPOSER_VERSION=1.9.3 # github-releases/composer/composer
ENV DOCKER_VERSION=19.03.1 # github-releases/docker/docker-ce&versioning=docker
ENV YARN_VERSION=1.19.1 # npm/yarn
```

The above (obviously not a complete `Dockerfile`, but abbreviated for this example), could then be supported accordingly:

```json
{
  "regexManagers": [
    {
      "fileMatch": ["^Dockerfile$"],
      "matchStrings": [
        "ENV .*?_VERSION=(?<currentValue>.*) # (?<datasource>.*?)/(?<depName>.*?)(\\&versioning=(?<versioning>.*?))?\\s"
      ],
      "versioningTemplate": "{{#if versioning}}{{versioning}}{{else}}semver{{/if}}"
    }
  ]
}
```

In the above the `versioningTemplate` is not actually necessary because Renovate already defaults to `semver` versioning, but it has been included to help illustrate why we call these fields _templates_. They are named this way because they are compiled using `handlebars` and so can be composed from values you collect in named capture groups.

By adding the comments to the `Dockerfile`, you can see that instead of four separate `regexManagers` being required, there is now only one - and the `Dockerfile` itself is now somewhat better documented too. The syntax we used there is completely arbitrary and you may choose your own instead if you prefer - just be sure to update your `matchStrings` regex.