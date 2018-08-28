# Tugboat Configuration File

Tugboat uses a file at `.tugboat/config.yml` in a git repository to configure
the previews it builds. Each preview is built from the `config.yml` file in its
own branch, tag, commit, or pull request. This file is required in order for
Tugboat to build any previews for a repository.

The only top-level key of the configuration file is `services`. It is a required
key, and is a map of the services required by a preview.

Every service consists of the service's name as the key, and a map as the value.
The name must be unique in the `services` list, and must also be a
[valid host name](https://en.wikipedia.org/wiki/Hostname#Restrictions_on_valid_hostnames)
containing only the characters `a-z`, `0-9`, and `-`.

The following attributes control how a service is built:

| Key                            | Type    | Description                                         |
| :----------------------------- | :------ | :-------------------------------------------------- |
| [image](#image)                | String  | The Docker image to use for this service            |
| [checkout](#checkout)          | Boolean | Whether to clone the git repository to this service |
| [checkout_path](#checkoutpath) | String  | Where to clone the git repository                   |
| [depends](#depends)            | List    | List of other services that this service depends on |
| [commands](#commands)          | List    | List of commands to run for various build stages    |

The following attributes configure how service URLs are generated

| Key                      | Type    | Description                                               |
| :----------------------- | :------ | :-------------------------------------------------------- |
| [aliases](#aliases)      | List    | A list of aliases to generate URLs for                    |
| [alias_type](#aliastype) | String  | What kind of aliases to generate                          |
| [subpath](#subpath)      | Boolean | Whether subpath URLs should be generated for this service |

The following attributes configure how the Tugboat proxy routes HTTP requests to
the service:

| Key                              | Type    | Description                                               |
| :------------------------------- | :------ | :-------------------------------------------------------- |
| [expose](#expose)                | Integer | Which port the service should expose to the Tugboat proxy |
| [default](#default)              | Boolean | Whether this is the default service for a preview.        |
| [http](#http)                    | Boolean | Whether the service should be available via HTTP          |
| [https](#https)                  | Boolean | Whether the service should be available via HTTPS         |
| [https_redirect](#httpsredirect) | Boolean | Whether HTTP requests should be redirected to HTTPS       |
| [domain](#domain)                | String  | A custom domain for Tugboat to generate URLs with         |
| [subpath_map](#subpathmap)       | Boolean | Whether to map the generated subpath to "/"               |

---

## `alias_type`

* **Type:** String
* **Default:** `default`
* **Required:** No

What type of URL aliases to generate for the service. Valid options are
`default` or `domain`. Alias URLs are generated in addition to the normal
Service URLs.

#### `default`

When `alias_type` is set to `default`, the alias URLs are constructed by
substituting the preview name in the Service URL with the values of `aliases`.
The values of `aliases` are sanitized based on the value of
[`subpath`](#subpath).

If `subpath` is `false`, alias values are sanitized to create a valid host name,
and is truncated to 30 characters, to make the host name a maximum of 63
characters. If `subpath` is `true`, the alias values are URL encoded.

If `aliases` is set to `['foo', 'bar']`, alias URLs look like the following

* https://foo-4vdrhxvyddvr5tne7zcr4y72vzowqohj.tugboat.qa
* https://bar-4vdrhxvyddvr5tne7zcr4y72vzowqohj.tugboat.qa

or

* https://previews.tugboat.qa/foo-4vdrhxvyddvr5tne7zcr4y72vzowqohj/
* https://previews.tugboat.qa/bar-4vdrhxvyddvr5tne7zcr4y72vzowqohj/

#### `domain`

When `alias_type` is set to `domain`, the alias URLs are constructing by
substituting the domain part of the Service URL with the values of `aliases`.
The alias values are sanitized to create a valid domain name. If `aliases` is
set to `['foo.com', 'bar.com']`, alias URLs look like the following

* https://pr123-4vdrhxvyddvr5tne7zcr4y72vzowqohj.foo.com
* https://pr123-4vdrhxvyddvr5tne7zcr4y72vzowqohj.bar.com

or

* https://foo.com/pr123-4vdrhxvyddvr5tne7zcr4y72vzowqohj/
* https://bar.com/pr123-4vdrhxvyddvr5tne7zcr4y72vzowqohj/

In order for these domains to resolve, a DNS entry must be added to the alias
domains as a `CNAME` to `previews.tugboat.qa`. A wildcard entry is required if
`subpath` is set to `false`.

---

## `aliases`

* **Type:** List
* **Default:** _no default_
* **Required:** No

A list of aliases to generate URLs for. If set, additional alias URLs will be
generated for the service. These URLs can be used to route to different
endpoints inside of the service, such as for a Drupal Multisite. How the alias
URLs are constructed depend on the value of [`alias_type`](#aliastype)

---

## `checkout`

* **Type:** Boolean
* **Default:** `false` (`true` when `default: true` is set)
* **Required:** No

Whether or not this service should have a copy of the git repository cloned into
it.

---

## `checkout_path`

* **Type:** String
* **Default:** `/var/lib/tugboat`
* **Required:** No

Specifies the path where the git repository should be cloned if `checkout: true`
is set. If this path already exists, the clone will fail.

---

## `commands`

* **Type:** List
* **Default:** _no default_
* **Required:** No

A set of commands to run during various points in a preview's lifecycle. These
commands are divided into the following stages. Each stage consists of a list of
commands. Each command is run in its own context, so things like changing
directories does not "stick" between commands. If that behavior is requried, an
external script should be called.

| Stage  | Description                                                                                            |
| :----- | :----------------------------------------------------------------------------------------------------- |
| init   | Commands that set up the basic preview infrastructure, such as required packages or tools              |
| update | Commands that import data or other assets into a service, such as a database or image files            |
| build  | Commands that build or generate the site, such as compiling Sass or running database updates from code |
| ready  | Commands that indicate that a service is "ready", such as checking for a listening TCP port            |

The `init`, `update`, and `build` stages are related as follows

* When a preview is created, the commands in `init` are run, followed by the
  commands in `update`, and finally the commands in `build`.

* When a preview is refreshed, the commands in `update` are run, followed by the
  commands in `build`.

* When a preview is created from a base preview, only the commands in `build`
  are run.

---

## `default`

* **Type:** Boolean
* **Default:** `false` (`true` if only one service is defined)
* **Required:** Yes, if more than one service is defined.

Whether this is the default service for a preview. The default service is where
incoming HTTP requests to the preview URL are routed. Setting this to true also
implies `expose: 80` and `checkout: true` unless those attributes are explicitly
set otherwise.

---

## `depends`

* **Type:** List
* **Default:** _no default_
* **Required:** No

Defines the order in which commands are executed between the defined services.
If one service has a dependency on another, it will wait for that service's
commands to run before running its own commands. If not set, there is no
guaranteed order in which services will execute their commands relative to other
services.

---

## `domain`

* **Type:** String
* **Default:** _no default_
* **Required:** No

A custom domain to use when generating URLs for the service. If `example.com` is
used, for example, URLs for the service will resemble
`http://pr123-token.example.com` or `http://preview.example.com/pr123-token/`,
depending on the other configuration values. Note that using a custom domain
will result in browsers issuing SSL/TLS certificate warnings when combined with
`https: true`.

---

## `expose`

* **Type:** Integer
* **Default:** _no default_ (`80` when `default: true` is set)
* **Required:** No

If this service should be publicly accessible via HTTP/HTTPS, this is the port
that the Tugboat Proxy will forward incoming requests to.

---

## `http`

* **Type:** Boolean
* **Default:** `true`
* **Required:** No

When `true`, this service's URL is publicly accessible via HTTP on port 80

---

## `https`

* **Type:** Boolean
* **Default:** `true`
* **Required:** No

When `true`, this service's URL is public accessible via HTTPS on port 443. A
preview's default URL will always use the HTTPS URL if both `http` and `https`
are `true`.

---

## `https_redirect`

* **Type:** Boolean
* **Default:** `true`
* **Required:** No

When `true`, HTTP requests are automatically redirected to HTTPS. This setting
only applies if both `http` and `https` are `true`.

---

## `image`

* **Type:** String
* **Default:** _no default_
* **Required:** Yes

The Docker image to use for this service. Tugboat maintains a set of images on
[Dockerhub](https://hub.docker.com/u/tugboatqa). These images all extend the
official docker images, and are configured to work well with Tugboat.

---

## `subpath`

* **Type:** Boolean
* **Default:** `false`
* **Required:** No

When `true`, the URL generated for this service will be a subpath of the root
preview domain instead of a subdomain. Using a subpath URL is not common, but
can solve problems with testing advertisements or using OAuth

| subpath | URL Example                                                        |
| :------ | :----------------------------------------------------------------- |
| `true`  | https://preview.tugboat.qa/pr123-4vdrhxvyddvr5tne7zcr4y72vzowqohj/ |
| `false` | https://pr123-4vdrhxvyddvr5tne7zcr4y72vzowqohj.tugboat.qa          |

---

## `subpath_map`

* **Type:** Boolean
* **Default:** `true`
* **Required:** No

When `true`, and `subpath: true` is set, URLs are rewritten by the Tugboat Proxy
to replace the preview-specific path with `/` before being forwarded to the
service. When `false`, URLs are passed through as-is.