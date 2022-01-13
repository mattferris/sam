Smart Asset Manager
===================

SAM (Smart Asset Manager) or just *sam*, can be used to manage web assets such as CSS and JavaScript files. It works on the concept of asset *packs*. Each *pack* defines what files are to be included, how to fetch the files, how files should be processed, and what the resulting *packed* file should be called.

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
# ls assets.pack.min.css
assets.pack.min.css
```

The resulting file is the pack name suffixed with `min.css`. You can define a custom suffix be specifying it in the pack file.

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

Finally, you can specify a post-processor to complete any final processing of an asset. A post-processor is defined for one or more file extensions. A post-processors can also be defined for all extensions using an asterisk (`*`). This is useful if you want to minify the assets. You can define as many post-processors as you want. They will be run in the order they are defined. Each post-processor is run for each asset. A post-processor will only run for assets that come after it.

```
post js /path/to/jslint
post css,sass /path/to/sassc --type=css
post * /path/to/minify
```

Taking all the above, the resulting pack file might look like so.

```
# assets.pack

# options
set suffix .smaller.css
post /path/to/minifier

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
```
