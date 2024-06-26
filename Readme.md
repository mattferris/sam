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
# sam pack assets.pack
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

If there was more then one asset on the remote host, you could use variables to
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
cached and the appropriate commit is checked out before accessing the asset. To
use git assets, use the `git` command.

```
# local repo
git /home/user/assets.git master css/remote.css css/more.css

# remote repo
git git@github.com:user/assets master css/remote.css css/more.css
```

The `git` command expects the location of the repository and the commit to use,
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
Contrast this with HTTPS assets where a newly published version of the remote
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

Sam includes a features to assist with bundling assets for versioned deployment.
These features allow you to create a file structure that is able to be
referenced by a *tag*. The *tag* will resolve to a file list, from which the
ultimate asset URL will be resolveable.

Sam tracks versions in a similar manner to git. All versioned assets are
stored in a *repository* stored under `.deploy`. The repository must first be
initialized, which should be completed from the root folder of your asset
collections.

```
# cd assets
# sam init
info: initialized deploy repo in ~user/assets
```

A single repository can handle multiple collections of assets. These collections
are referred to as *projects*. A *project* lets you organize and version assets
independent of each other. Projects can be added and removed as necessary. When
a repository is initialized, a `default` project is created automatically.

```
# sam project --add main
info: created project main
```

As with git, `sam` uses a workflow that involves staging a number of files, then
commiting them. Each commit is identified by an ID (a SHA1 hash). When a commit
is created, two tags are automatically created which refer to the commit:
`latest` which always refers to the last commit, and `previous` which always
refers to the second last commit (if available).

```
# sam add --project main foo.css
# sam commit --project main
info: committed as c04e858b003cefb5642903f0495f70f6c17de25e
# sam tag --project main
latest -> c04e858b003cefb5642903f0495f70f6c17de25e
```

As `latest` and `previous` are transient tags, it is discouraged to rely on them
in production settings. Instead, additional tagging schemes should be used to
create more permanent references to specific commits.

To avoid having to specify the project for each command, you can specify the
default project to use.

```
# sam project
* default
  main
# sam project --use main
# sam project
  default
* main
```

In order to publish the versioned assets, simply copy the `.deploy` folder to
a web accessible location.

```
# cp -r .deploy /var/www/assets
```

To copy remotely, use `scp` or `rsync` instead.

```
# scp -r .deploy example.com:/var/www/assets
# rsync .deploy example.com:/var/www/assets
```

Alternatively, `sam` provides a `push` command which can simplify deployment
to remote servers or services. First a *remote* must be defined. At this time,
*remote*s must be defined manually.

To create a *remote* for OVH object storage, we would create a new file under
`.deploy/remotes` called `ovh`.

```
env /home/user/.config/sam/deploy.env
dest common/ver
push swift --os-auth-url {OS_AUTH_URL} --auth-version 3 --os-tenant-name {OS_TENANT_NAME} --os-project-name {OS_TENANT_NAME} --os-username {OS_USERNAME} upload --header 'Content-Type: {DEPLOY_MIME}' --object-name {DEPLOY_DEST} Static {DEPLOY_SRC}
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

Each line defines a name and value pair, separated by a space.

Once defined, you can use the `push` command to deploy your assets automatically.

```
# sam push ovh
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

By default, `sam` will only push the changes since the last commit. It's
possible to push all the objects using the `--full` option. The `--diff` option
specifies what tag/commit to base the changes from. The `--ref` option specifies
what tag/commit to push.

```
# sam push --full ovh
...
# sam push --ref v2.1 --diff v2.0 ovh
...
```

### Importing versioned assets

In some cases, you may need to import versioned assets into a repository. The
`pull` command accepts a tag to resolve with the specified *remote*. Once the
tag is resolved to a commit, `pull` will copy the associated objects from the
remote into the local repository, skipping objects that already exist. In
order to work with these objects, they must be incorporated into the working
tree of files. This is accomplished using the `checkout` command.

```
# ls
main.css
# sam pull ovh latest
# sam checkout latest
# ls
main.css foo.css bar.css
```

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

### Garbage Collection

Over time, a repository will come to hold more and more orphaned objects. These
are objects that are no longer referenced by any tags, and are therefore no
longer needed. Periodically pruning these objects will reduce the amount of
space the repository consumes on disk. Sam includes a garbage collection
command called `gc` which automatically prunes orphaned objects.

```
# sam gc
info: pruning object d25f87bf6a63b42b65bcaf1d7b444dead64e07e0
info: pruning object d43bd43d21df66c9b678eec9dee3b113212a29b4
info: pruning object de06f5317fa199730c7c0dbc8e0586443c1a9cd0
info: pruned 3 objects, 4672917 bytes reclaimed
```

Deployed assets (essentially, a remote repository) will also experience a growing
number of orphaned objects over time. This same command can be run on the remote
repository as well.

```
web-server# SAM_TREE_DIR= SAM_REPO_DIR=/var/www/assets sam gc
info: pruning object bd62ccf770ad138a4c29ca9e6e82c594075f707a
info: pruning object f1810e6f1cbefbe510c377807818c968533028d8
info: pruning object 174041ba1866b80aadc546a6a76116dbfc28aaea
info: pruning object 176d04e5de45a478821f2907a2d2fa4b90a083bc
info: pruning object 17ccd19fe28c5b8dba6dcce0d69faa6382f2515b
info: pruning object 17cd913ffd517133d60cf75d18cb7b7accdeb38d
info: pruning object 54ea8825de874a9486adbb6a713d123402680f95
info: pruning object 921c444d4aae1796b66e8df83384416a9a9f86a8
info: pruning object 92eabe228991742958bc7f1368f80e9f39619316
info: pruning object e226dd4cc024b903277dd31e8019375a60faa40a
info: pruned 10 objects, 2750734 bytes reclaimed
```

`sam` expects to be run in a working tree. In the above example, the environment
variables `SAM_TREE_DIR` and `SAM_REPO_DIR` are set so sam doesn't attempt to
traverse the filesystem looking for a repository stored in `.deploy`. This allows
deploy to run in a "bare" repository that has no working tree.

In cases where object storage is used, you may need to mount the container using
a command like `rclone` first.

```
# rclone mount ovh:container /mnt
# SAM_TREE_DIR= SAM_REPO_DIR=/mnt/assets sam gc
```
