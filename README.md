# Sylius plugin

A **development-only dependency pack** for [Sylius](https://sylius.com) plugin authors.

Installing this single package pulls in the curated tooling stack used to build Sylius plugins — static analysis (PHPStan with Sylius-aware stubs), tests (PHPUnit, Prophecy, Infection), refactoring (Rector), coding standard, and more — so individual plugins don't each have to pin and upgrade those tools themselves.

Install it as a dev dependency in your Sylius plugin, and pin the tag that matches the Sylius version you are targeting:

```shell
composer require --dev setono/sylius-plugin
```
