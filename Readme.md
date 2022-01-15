Smart Asset Manager
===================

SAM (Smart Asset Manager) or just *sam*, can be used to manage web assets such as CSS and JavaScript files. It works on the concept of asset *packs*. Each *pack* defines what files are to be included, how to fetch the files, how files should be processed, and what the resulting *packed* file should be called.

Defining assets
---------------

Most lines in a *pack* file represent one or more assets to include. A simple asset pack might just be used to assemble a number of CSS files into a single file. Lines starting with `#` are comments.

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

You can define a custom suffix by specifying it in the pack file. The default suffix is `.out`.

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

Options can be set using the `set` command. Similarily, `def` defines a variable. Variables can be useful to save yourself some typing.

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

It's possible to include remote assets, too. Sam supports accessing assets using HTTPS. Remotely accessed assets are cached locally, and the cached copy will be used until it is cleared from the cache. This improves performance, but also prevents issues that arise from the remote file not being accessible.

Referencing a remote asset using HTTPS is pretty simple.

```
https://www.example.com/assets/css/remote.css
```

If there was more then once asset on the remote host, you could use variables to make our life easier.

```
def remote_host https://www.example.com/assets/css

{remote_host}/remote.css
{remote_host}/more.css
```

You can also specify a SHA1 checksum for the asset, in which case the asset won't be included unless it matches the supplied checksum. The full checksum is not required, merely a prefix of one or more characters It's recommended that you specify a minimum of four characters, though seven or eight characters is likely good enough for most situations.

```
{remote_host}/remote.css 987be93a
```

Git assets
----------

Files can also be included using git. This works for local and remote repositories. In both cases, the referenced repository is cloned into the local cached, and the appropriate commit is checked out, before accessing the asset. To use git assets, the `git` command used.

```
# local repo
git /home/user/assets.git master css/remote.css css/more.css

# remote repo
git git@github.com:user/assets master css/remote.css css/more.css
```

The git command expects the location of the repository and the commit to use, followed by one or more assets. It's also possible to specify the assets on subsequent (indented) lines.

```
git /home/user/assets.git master
  css/remote.css
  css/more.css
```

Using git to reference assets can be nice because you can pin the included asset to a particular version by using a commit ID. Even if the repo is updated, so long as the commit exists, you'll always get the same version of the asset.

```
# pin remote.css to a specific commit
git git@github.com:user/assets 87dfba2 css/remote.css
```

Git assets are the most robust mechanism for referencing remote assets. Referencing assets by a specific commit (rather then branch name) ensures you'll always receive the same asset content. Referencing by commit ID also means the remote repository can continue to be updated without affecting the assets. Contrast this with HTTPS assets, where a newly published version of the remote asset will break the building of the pack if the asset isn't cached.

Importing assets from another pack
----------------------------------

It's possible to import to assets from another pack file. The imported pack file is processed normally, except that any post-processors defined in the imported pack are ignored. Relative paths to assets are resolved to absolute paths before being included in the importing pack.

Use `import` to import assets.

```
import /some/other/assets.pack
```

One advantage of using import is that remote assets in the imported pack will be resolved to the imported packs remote cache, preventing the assets from being fetched and stored in the importing packs remote cache.

Post-processors
---------------

Finally, you can specify a post-processor to complete any final processing of an asset. A post-processor is defined for one or more file extensions. A post-processors can also be defined for all extensions using an asterisk (`*`). This is useful if you want to minify the assets. You can define as many post-processors as you want. They will be run in the order they are defined. Each post-processor is run for each asset. A post-processor will only run for assets that come after it.

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
