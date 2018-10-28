# Advanced composer setup

---

**NOTICE**: You need to think about this *only* if your plugin depends on third party composer packages. 

---


People will want to install your plugin with two methods: 
1. downloading a zip
2. requiring it with composer. 

If your plugin depends on other composer packages the two targets require different distributions. 
People installing with composer will download your dependencies automatically and store them in their own vendor folder,
while people without composer want to download a simple zip with all dependencies included. 

## Problem 1: the vendor folder

The [general recommendation](https://getcomposer.org/doc/faqs/should-i-commit-the-dependencies-in-my-vendor-directory.md) 
is to not include vendor folders in repositories. Therefore it should be included only in distribution zips 
(for the people that don't use composer). 

However, a vendors folder included in a zip should not contain `getkirby/cms` since Kirby will already be installed in the project root. 
A script that creates the distribution zip should therefore temporarily remove `getkirby/cms` from the requirements. 

A CircleCI job might look like this:

```yaml
jobs:
  build:
    docker:
      - image: circleci/php:7.2.9-cli-stretch-node

    working_directory: ~/repo

    steps:
      - checkout
      
      - run: composer 
      
      # Remove getkirby/cms from dependencies
      - run:
        name: Update composer.json
        command: composer remove "getkirby/cms" --no-update

      # Download and cache dependencies
      - restore_cache:
          keys:
          - v1-dependencies-{{ checksum "composer.lock" }}
          # fallback to using the latest cache if no exact match is found
          - v1-dependencies-
          
      - run: 
        name: Install vendor libraries
        command: composer update --no-dev

      - save_cache:
          paths:
            - vendor
          key: v1-dependencies-{{ checksum "composer.lock" }}

      - run:
          name: Create zip
          command: |
            mkdir ./artifacts
            zip artifacts/build -r -x '*artifacts/*'

      - persist_to_workspace:
          root: ./artifacts
          paths:
            - build.zip
```

## Problem 2: Who loads your dependencies?

Composer users load your dependencies automatically, while zip users rely on your plugin to load them. 
In this setup you need to alter your `autoload` property in `composer.json` like this:

```json
{
  "name": "yourname/reponame",
  "description": "This description will be visible on packagist",
  "authors": [
    {
      "name": "Your name",
      "email": "your@email.address"
    }
  ],
  "require": {
    "getkirby/cms": "*"
  },
  "license": "ISC",
  "autoload": {
    "files": [
      "config.php"
    ]
  }
}
```

`config.php` will contain the actual plugin code, while `index.php` only requires the zipped composer autoloader:
```php
<?php

require_once __DIR__ . '/vendor/autoload.php';

```
When installed via composer, `config.php` will be loaded straight away (index.php is simply ignored). 

While a zip install will take a detour: kirby by default loads index.php; index.php loads the autoloader; and the autoloader
will honor the plugins composer.json instructuction to load `config.php`, after setting up your dependencies.

Another approach to this problem would be to `require_once` your vendor folder conditionally in your normal index.php:

```php
<?php
// my composer.json points to index.php! 

if(!class_exists('ThirdPartyClass')) {
  require_once __DIR__ . '/vendor/autoload.php';
}

Kirby::plugin(
  //continue as usual
);
```
