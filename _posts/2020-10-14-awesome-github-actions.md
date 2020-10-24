---
layout: post
title: "Awesome Github Actions"
description: "Try Github Actions with Node.js and Python projects."
categories: [github, node.js, python]
comments: true
---

{% raw %}

## Deploy Github Pages with Github Actions

Just weeks ago, I set up this blog with Jekyll on Github Pages. However, I switched to [fastpages](https://github.com/fastai/fastpages) because of Jupyter Notebook support. The swithcing isn't that difficult, but something caught my attention during the process, and that is [Github Actions](https://github.com/features/actions). We have talked about the idea that only pushing source files to the Github repo and let Github Pages generate static files by its own. But Github Pages only supports Jekyll and limited number of plugins, and that is where Github Actions can help.

Basically, Github Actions is like a remote server that can execute some predifined commands. To use Github Actions to deploy our Github Pages, we still need to generate static files in our repo, but the generation can be done automatically by Github Actions, and generated files can be hosted in another branch other than master. Therefore, we can host all source files in the master branch, and every time we push commits to the master branch (e.g. new posts or modifications of layouts), the predifined Github Actions workflow is triggered and it will automatically generate all necessary static files in another branch. We just need to change the repo's setting that decides which branch Github Pages site is built from.

Take [my homepage repo](https://github.com/graysonliu/graysonliu.github.io) as an example. I build it using React and Webpack from scratch. Webpack will generate static files in a folder called `dist`. This dist folder does not have to be in the master branch, but all files inside it should be the content of another branch called `gh-pages`, and the Github Pages is built from this branch.

There are some Github Actions in the marketplace to help us achieve that for Github Pages. I chose the one that fastpages uses, which is [peaceiris/actions-gh-pages](https://github.com/peaceiris/actions-gh-pages). The usage is actually pretty straightforward, the entire workflow is as follows:
```yaml
name: build

on:
  push:
    branches:
      - master

jobs:
  deploy:
    runs-on: ubuntu-18.04
    steps:
      - uses: actions/checkout@v2

      - name: Setup Node
        uses: actions/setup-node@v2.1.0
        with:
          node-version: '12.x'

      - name: Cache dependencies
        uses: actions/cache@v2
        with:
          path: ~/.npm
          key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
          restore-keys: |
            ${{ runner.os }}-node-
      - run: npm ci
      - run: npm run build

      - name: Deploy
        uses: peaceiris/actions-gh-pages@v3
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./dist
```

At first, we give a name to the workflow, which is `build`. Then, we specify what will trigger this workflow:

```yaml
on:
  push:
    branches:
      - master
```
That is, every time we push commits to the master branch, this workflow will be executed. Now, we define all commands in this workflow. We first specify the operating system, checkout the repo and setup Node.js of the version we want. The following action `actions/cache@v2` is to fetch all cached dependencies using `key`, or create the cache of all dependencies if the cache does not exist, so that the workflow does not need to download all dependencies again next time. In Linux, `npm` downloads all packages to `~/.npm`, which is the folder we need to cache. The `key` of the cache is named after the OS and the hashing result of `package-lock.json`, which is reasonable since these two will decide the packages that npm downloads.

Now, we run bash command `npm ci` to download all dependencies (if one dependency exists in cache, `npm` will just use the alreay downloaded package, while new dependencies will be downloaded from the Internet). Then, we run `npm run build` to generate all static files. In my [package.json](https://github.com/graysonliu/graysonliu.github.io/blob/master/package.json), `npm run build` will use Webpack to achieve that:
```json
{
  ...
  "scripts": {
      "start": "webpack-dev-server --mode development",
      "build": "webpack  --mode production"
    }
  ...
}
```
Webpack generate all files in the `/dist` folder, and we should deploy all content in this folder to the branch `gh-pages`. That is excatly what `peaceiris/actions-gh-pages` does at last. As for `${{ secrets.GITHUB_TOKEN }}`, this is an automactically generated token by Github for authentication in the workflow, since this action needs to be authorized to write into our repository. We will find more usages of secrets in the next part.

## Periodically Spotify Playlists Updating with Github Actions

I once wrote a Python script that can update my Spotify top 200 playlists with data from [SpotifyCharts](https://spotifycharts.com). To update playlists periodically, this script will be executed everyday on my own server. However, with Github Actions, we can schedule a workflow that runs the script at any time. The repo is at [graysonliu/spotify-top-200-playlist-generator](https://github.com/graysonliu/spotify-top-200-playlist-generator). We use this [workflow](https://github.com/graysonliu/spotify-top-200-playlist-generator/blob/master/.github/workflows/python.yml) to update the playlists daily. At the beginning, apart from push-triggering, this workflow is also triggered at 00:00, 06:00, 12:00 and 18:00 every day:

```yaml
on:
  push:
    branches:
      - master
  schedule:
    # * is a special character in YAML so you have to quote this string
    - cron: '0 0,6,12,18 * * *'
```

Similar to the previous Node.js project, we also need to checkout the repo, specify the version of operating system and Python, cache and install all dependencies that pip downloads:
```yaml
      - uses: actions/checkout@v2
      - name: Set up Python 3.6
        uses: actions/setup-python@v2
        with:
          python-version: 3.6

      - uses: actions/cache@v2
        with:
          path: ~/.cache/pip
          key: ${{ runner.os }}-pip-${{ hashFiles('**/requirements.txt') }}
          restore-keys: |
            ${{ runner.os }}-pip-
      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          if [ -f requirements.txt ]; then pip install -r requirements.txt; fi
```

Then, we execute the Python script to update our playlists. When we run this script locally, we can set client id and client secret of our Spotify app as environment variables in `.env`, and their values can be obtained in Python script by simply reading environment variables. However, sensitive information like client id and client secret should not be exposed, meaning that we should not host this `.env` file on Github. Therefore, we add them as secrets at Github, and set them as environment variables in the workflow (actually, client id is not sensitive at all, you can just put it in the `config.yml`).

Modifying playlists in Spotify needs authentication, and that requires a browser. However, since Github Actions works in a headless environment, it is impossible to use a browser for authentication. Our strategy is, we first authenticate the app locally, which will give us a `.cache` file that saves tokens and can be used for authentication for a long period of time. We create secret `AUTH_CACHE` that saves the content of `.cache` at Github and also set it as a environment variable in the runtime. In the workflow, when we execute the Python script, we fetch this environment variable and create a `.cache` file using this secret. This newly created `.cache` file will be used for authentication.

I tried to create `.cache` by `echo` command in the workflow directly:

```yaml
run: echo ${{ secrets.SPOTIFY_AUTH_CACHE }} >> .cache
```

This will not work because `AUTH_CACHE` is a JSON string that has quotation marks inside it, but echo somehow omits those quotation marks. So, we as well use Python to achieve that:

```python
auth_cache = os.getenv('AUTH_CACHE')
if auth_cache:
    with open('.cache', 'w') as f:
        f.write(auth_cache)
```

Another thing is, tokens in `.cache` could be refreshed in the runtime. If we do not update `AUTH_CACHE` to keep up with the content of `.cache`, tokens saved in `AUTH_CACHE` could be expired. Therefore, after the playlists are updated, we have to write the content of `.cache` back to secret `AUTH_CACHE` in case file `.cache` changes (though it seems it never changes according to my experience). We have to use Github APIs to update secrets in a workflow, but that needs authentication from Github. The default `${{ secrets.GITHUB_TOKEN }}` does not have the permission to write secrets. For that purpose, we need a token with special scopes of permissions. First, we create a personal access token named `TOKEN_WRITE_SECRETS` with scopes as follows:

![]({{ site.baseurl }}/images/create_personal_access_token.png)

Copy the generated token, and add it to secrets. We also name this secret `TOKEN_WRITE_SECRETS`.

In total, we should have four secrets now:

![]({{ site.baseurl }}/images/secrets.png)

After the updating is finished, we write the content of `.cache` back to `AUTH_CACHE` in the Python script. To write a secret, we should encrypt its content using the repo's public key first. This public key can also be acquired by Github APIs (see [this](https://docs.github.com/en/free-pro-team@latest/rest/reference/actions#get-a-repository-public-key)):

```python
# update secret AUTH_CACHE in case that content in .cache is changed during runtime (e.g. token is refreshed)
secret_name = 'AUTH_CACHE'
# we need authentication to write secrets
# default GITHUB_TOKEN does not have the permission to write secrets, we need to create a personal access token
# we also need to set this personal access token as an environment variable
token_write_secrets = os.getenv('TOKEN_WRITE_SECRETS')

# these are default environment variables in Github Actions
github_api_url = os.getenv('GITHUB_API_URL')
github_repo = os.getenv('GITHUB_REPOSITORY')
github_actor = os.getenv('GITHUB_ACTOR')

# for authentication
from requests.auth import HTTPBasicAuth

auth = HTTPBasicAuth(github_actor, token_write_secrets)

headers = {'Accept': 'application/vnd.github.v3+json'}
# Get the public key to encrypt secrets
# reference: https://docs.github.com/en/free-pro-team@latest/rest/reference/actions#get-a-repository-public-key
r = requests.get(f'{github_api_url}/repos/{github_repo}/actions/secrets/public-key', headers=headers, auth=auth)
public_key = r.json()['key']
key_id = r.json()['key_id']
```
Then, we invoke the Github API that modifies secrets (see [this](https://docs.github.com/en/free-pro-team@latest/rest/reference/actions#create-or-update-a-repository-secret)):

```python
# reference: https://docs.github.com/en/free-pro-team@latest/rest/reference/actions#create-or-update-a-repository-secret
from base64 import b64encode
from nacl import encoding, public


def encrypt(public_key: str, secret_value: str) -> str:
    """Encrypt a Unicode string using the public key."""
    public_key = public.PublicKey(public_key.encode("utf-8"), encoding.Base64Encoder())
    sealed_box = public.SealedBox(public_key)
    encrypted = sealed_box.encrypt(secret_value.encode("utf-8"))
    return b64encode(encrypted).decode("utf-8")


with open('.cache', 'r') as f:
    encrypted_value = encrypt(public_key, f.read())
    data = {'encrypted_value': encrypted_value, 'key_id': key_id}
    r = requests.put(f'{github_api_url}/repos/{github_repo}/actions/secrets/{secret_name}', headers=headers, json=data, auth=auth)
    if r.ok:
        print(f'Secret {secret_name} updated.')
```

Run the entire Python script in the workflow with four secrets we defined before:

```yaml
      - name: Set environment variables with secrets and update playlists
        env:
          CLIENT_ID: ${{ secrets.CLIENT_ID }}
          CLIENT_SECRET: ${{ secrets.CLIENT_SECRET }}
          AUTH_CACHE: ${{ secrets.AUTH_CACHE }}
          TOKEN_WRITE_SECRETS: ${{ secrets.TOKEN_WRITE_SECRETS }}
        run: python generate_top_200_playlist.py
```

Now, this workflow will update Spotify playlists at scheduled times every day. It uses `AUTH_CACHE` for authentication for Spotify, and uses a personal access token `TOKEN_WRITE_SECRETS` for authentication for Github APIs that help us to write the content of `.cache` back to secret `AUTH_CACHE`.

At last, we remove `.cache` since its content has already been saved in secret `AUTH_CACHE`:

```yaml
      - name: Delete local authorization cache file
        run: rm -f .cache
``` 

This is mainly for security reasons, though it might not be necessary since this file was created in the workflow and its lifecycle should end in the current workflow run.

{% endraw %}