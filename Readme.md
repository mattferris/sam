Smart Asset Manager
===================

SAM (Smart Asset Manager) or just *sam*, can be used to manage web assets such
as CSS and JavaScript files. It works on the concept of asset *packs*. Each
*pack* defines what files are to be included, how to fetch the files, how files
should be processed, and what the resulting *packed* file should be called.

Defining assets
---------------

Most lines in a *pack* file represent one or more assets to include. A simple
asset pack might just be used to assemble a number of CSS files into a single
file. Lines starting with `#` are comments.

```
# assets.pack
form.css
font.css
layout.css
page.css
reset.css
```

Run sam to package the assets.

```
# ./sam assets.pack
# ls assets.pack.out
assets.pack.out
```

Note that any duplicate asset paths are ignored.

Options and variables
---------------------

You can define a custom suffix by specifying it in the pack file. The default
suffix is `.out`.

```
# assets.pack

# options
set suffix .smaller.css

# assets
form.css
font.css
layout.css
page.css
reset.css
```

Options can be set using the `set` command. Similarily, `def` defines a variable.
Variables can be useful to save yourself some typing.

```
# assets.pack

# options
set suffix .smaller.css

# define path to CSS files
def path /home/user/css

# assets
{path}/form.css
{path}/font.css
{path}/layout.css
{path}/page.css
{path}/reset.css
```

### Available options

| Name   | Default          | Description                      |
|--------|------------------|----------------------------------|
| suffix | `.out`           | The suffix of the resulting file |
| target | `{pack}{suffix}` | The name of the resulting file   |
| mode   | `0644`           | The mode of the resulting file   |
| owner  | *null*           | The owner of the resulting file  |
| group  | *null*           | The group of the resulting file  |

Remote assets
-------------

It's possible to include remote assets, too. Sam supports accessing assets using
HTTPS. Remotely accessed assets are cached locally, and the cached copy will be
used until it is cleared from the cache. This improves performance, but also
prevents issues that arise from the remote file not being accessible.

Referencing a remote asset using HTTPS is pretty simple.

```
https://www.example.com/assets/css/remote.css
```

If there was more then once asset on the remote host, you could use variables to
make your life easier.

```
def remote_host https://www.example.com/assets/css

{remote_host}/remote.css
{remote_host}/more.css
```

You can also specify a SHA1 checksum for the asset, in which case the asset won't
be included unless it matches the supplied checksum. The full checksum is not
required, merely a prefix of one or more characters. It's recommended that you
specify a minimum of four characters, though seven or eight characters is likely
good enough for most situations.

```
{remote_host}/remote.css 987be93a
```

Git assets
----------

Files can also be included using git. This works for local and remote
repositories. In both cases, the referenced repository is cloned into the local
cached, and the appropriate commit is checked out before accessing the asset. To
use git assets, use the `git` command.

```
# local repo
git /home/user/assets.git master css/remote.css css/more.css

# remote repo
git git@github.com:user/assets master css/remote.css css/more.css
```

The git command expects the location of the repository and the commit to use,
followed by one or more assets. It's also possible to specify the assets on
subsequent (indented) lines.

```
git /home/user/assets.git master
  css/remote.css
  css/more.css
```

Using git to reference assets can be nice because you can pin the included asset
to a particular version by using a commit ID. Even if the repo is updated, so
long as the commit exists, you'll always get the same version of the asset.

```
# pin remote.css to a specific commit
git git@github.com:user/assets 87dfba2 css/remote.css
```

Git assets are the most robust mechanism for referencing remote assets.
Referencing assets by a specific commit (rather then branch name) ensures you'll
always receive the same asset content. Referencing by commit ID also means the
remote repository can continue to be updated without affecting the assets.
Contrast this with HTTPS assets, where a newly published version of the remote
asset will break the building of the pack if the asset isn't cached.

Importing assets from another pack
----------------------------------

It's possible to import to assets from another pack file. The imported pack file
is processed normally, except that any post-processors defined in the imported
pack are ignored. Relative paths to assets are resolved to absolute paths before
being included in the importing pack.

Use `import` to import assets.

```
import /some/other/assets.pack
```

One advantage of using import is that remote assets in the imported pack will be
resolved to the imported packs remote cache, preventing the assets from being
fetched and stored in the importing packs remote cache.

Post-processors
---------------

Finally, you can specify a post-processor to complete any final processing of an
asset. A post-processor is defined for one or more file extensions. A
post-processor can also be defined for all extensions using an asterisk (`*`).
This is useful if you want to minify the assets. You can define as many
post-processors as you want. They will be run in the order they are defined. Each
post-processor is run for each asset. A post-processor will only run for assets
that defined after it in the pack file.

```
post js /path/to/jslint
post css,sass /path/to/sassc --type=css
post * /path/to/minify
```

Putting it all together
-----------------------

Taking all the above, the resulting pack file might look like so.

```
# assets.pack

# options
set suffix .smaller.css
post css /path/to/minifier

# path to local assets
def path /home/user/assets
def repo git@github.com:user/assets

# remote assets
git {repo} master
  css/remote.css
  css/more.css

# local assets
{path}/form.css
{path}/font.css
{path}/layout.css
{path}/page.css
{path}/reset.css

# import common assets from another pack
import common.pack
```

Versioned asset deployment
--------------------------

Sam includes a utility to assist with bundling assets for versioned deployment
creatively called `deploy`. `deploy` will create a file structure that is able
to be referenced by a *tag*. The *tag* will resolve to a file list, from which
the ultimate asset URL will be resolveable.

`deploy` tracks versions in a similar manner to git. All versioned assets are
stored in a *repository* stored under `.deploy`. The repository must first be
initialized, which should be completed from the root folder of your asset
collections.

```
# cd assets
# deploy init
info: initialized deploy repo in ~user/assets
```

A single repository can handle multiple collections of assets. These collections
are referred to as *projects*. A *project* lets you organize and version assets
independent of each other. Projects can be added and removed as necessary. When
a repository is initialized, a `default` project is created automatically.

```
# deploy project --add main
info: created project main
```

As with git, `deploy` uses a workflow that involves staging a number of files,
then commiting them. Each commit is identified by an ID (a SHA1 hash). When a
commit is created, two tags are automatically created which refer to the commit:
`latest` which always refers to the last commit, and a timestamp of when the
commit was created (e.g. `2023.05.29-16:30:20`).

```
# deploy add --project main foo.css
# deploy commit main
info: committed as c04e858b003cefb5642903f0495f70f6c17de25e
# deploy tag --project main
2023.05.29-16:30:20 -> c04e858b003cefb5642903f0495f70f6c17de25e
latest -> c04e858b003cefb5642903f0495f70f6c17de25e
```

In order to publish the versioned assets, simply copy the `.deploy` folder to
a web accessible location.

```
# cp -r .deploy /var/www/assets
```

To push remotely, use `scp` or `rsync` instead.

```
# scp -r .deploy example.com:/var/www/assets
# rsync .deploy example.com:/var/www/assets
```

Alternatively, `deploy` provides a `push` command which can simplify deployment
to remote servers or services. First a *remote* must be defined. At this time,
*remote*s must be defined manually.

To create a *remote* for OVH object storage, we would create a new file under
`.deploy/remotes` called `ovh`.

```
env /home/user/.config/sam/deploy.env
dest common/ver
exec swift --os-auth-url {OS_AUTH_URL} --auth-version 3 --os-tenant-name {OS_TENANT_NAME} --os-project-name {OS_TENANT_NAME} --os-username {OS_USERNAME} upload --header 'Content-Type: {DEPLOY_MIME}' --object-name {DEPLOY_DEST} Static {DEPLOY_SRC}
```

The first line defines a file to read environment variables from. The second line of the *remote* file defines a destination folder. The third line, defines the command to execute for each file that needs to be deployed. The curly braces are used to reference environment variables.

The contents of the environment file looks like so.

```
OS_AUTH_URL https://auth.cloud.ovh.net/v3
OS_TENANT_ID 09809e7c712312a09723879e987987cb1983
OS_TENANT_NAME 908230498029374873
OS_USERNAME user-09234oijwdf0
OS_PASSWORD 098u0o2i3jloaisudpf9u
OS_REGION_NAME BHS
```

Each line defined a name and value pair, separated by a space.

One defined, you can use the `push` command to deploy your assets automatically.

```
# deploy push --project main ovh
info: pushing objects/d7/d76009f3e8533dcee64ea80fd671ba665b4cfb82
info: pushing objects/c8/c8f9738701a05e1ed83aac8e958798c2524eee2d
info: pushing objects/96/969efc7579f3b515a419c739d701d3fdb766336d
info: pushing objects/3b/3b959ae62204e69666984dd0bc125c6ce8ed4956
info: pushing projects/identity/tags/latest
```

Not only does `push` automate the method of deployment, it also ensures that
each deployment is atomic. Tags will not be deployed until the entire commit
has been deployed first. Without this consideration, a tag would reference a
commit that doesn't exist, or assets that are still in the process of being
deployed.

### How to reference a versioned asset

In order to resolve a versioned asset, a server-side or client-side code snippet
will be required to handle the resolution of the asset to a URL, which can then
be used to refer to the asset.

Let's look at a versioned asset URL.

```
https://www.example.com/assets/theme/page.css:latest
```

The above URL can be broken down into four parts.

- The *repository* URL (`https://www.example.com/assets`)
- The *project* name (`theme`)
- The *asset* name (`page.css`)
- The version *tag* (`latest`)

The steps involved in resolving the versioned URL to a specific asset are:

- Request the contents of the *tag* which will return the *commit* ID
- Request the *commit* which will return a list of *assets* (and corresponding IDs)
- Resolve the URL of the asset by appending the asset path to the *repository* URL

In practice, this would look like:

```
# curl https://www.example.com/assets/projects/tags/latest
17796fc6bdf1a285ffd5e61c444b89d25510efc0

# curl https://www.example.com/assets/objects/17/17796fc6bdf1a285ffd5e61c444b89d25510efc0
page.css f1886a5a91998d12ca99e7fb1f1bb06143d89d5b

# curl https://www.example.com/assets/objects/f1/f1886a5a91998d12ca99e7fb1f1bb06143d89d5b
.page {
    margin: 10px;
}
```

Ideally, the result would be cached for a certain amount of time.
