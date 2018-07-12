# [WIP] Sitemap bundle

[![Latest Stable Version](http://img.shields.io/packagist/v/jeroendesloovere/sitemap-bundle.svg)](https://packagist.org/packages/jeroendesloovere/sitemap-bundle)
[![License](http://img.shields.io/badge/license-MIT-lightgrey.svg)](https://github.com/jeroendesloovere/sitemap-bundle/blob/master/LICENSE)
[![Build Status](https://travis-ci.org/jeroendesloovere/sitemap-bundle.svg)](https://travis-ci.org/jeroendesloovere/sitemap-bundle)
[![Scrutinizer Code Quality](https://scrutinizer-ci.com/g/jeroendesloovere/sitemap-bundle/badges/quality-score.png)](https://scrutinizer-ci.com/g/jeroendesloovere/sitemap-bundle/)

> This Symfony bundle allows you to easily generate a sitemapindex and one or multiple sitemap(s).

## Work In progress

What needs to be done:
* Generate the "sitemap provider last modified on datetime" which we need to show in the sitemapindex.

## Usage

### Installation

```bash
composer require jeroendesloovere/sitemap-bundle
```

### Example: "How to create your custom sitemap?"

We need to notify Symfony that we have a new sitemap provider.
Add the following somewhere in your `services.yaml`
```yaml
services:
    App\SitemapProviders\NewsArticleSitemapProvider:
        tags:
            - { name: sitemap.provider }
```

When the `SitemapGenerator` needs to generate the sitemap(s),
it will ask all SitemapProviders to fill in the items.
Create something like the following in your app.
```php
<?php

namespace App\SitemapProviders;

use JeroenDesloovere\SitemapBundle\Item\ChangeFrequency;
use JeroenDesloovere\SitemapBundle\Provider\SitemapProvider;
use JeroenDesloovere\SitemapBundle\Provider\SitemapProviderInterface;

class NewsArticleSitemapProvider extends SitemapProvider implements SitemapProviderInterface
{
    /** @var NewsArticleRepository */
    private $articleRepository;

    public function __construct(NewsArticleRepository $articleRepository)
    {
        $this->articleRepository = $articleRepository;

        // `NewsArticle::class` would even be better then just `NewsArticle`
        // because you can then use it with doctrine events.
        parent::__construct('NewsArticle');
    }

    public function createItems(): void
    {
        /** @var Article[] $articles */
        $articles = $this->articleRepository->findAll();
        foreach ($articles as $article) {
            $this->createItem('/nl/xxx/url-to-article', $article->getEditedOn(), ChangeFrequency::monthly());
        }
    }
}
```

You can now generate the sitemap(s) by executing:
```bash
bin/console sitemap:generate
```
> Use a cronjob (f.e. every hour) to have up-to-date sitemaps.

OR if you want to use PHP
```php
$this->getContainer()->get('sitemap.generator')->generate();
```

### Another example: "How to use a Doctrine subscriber?"

Sometimes you want to fetch database changes from Doctrine events.
The following code helps you doing that and regenerates the sitemaps.

```yaml
    sitemap.subscriber:
        class: JeroenDesloovere\SitemapBundle\EventListener\SitemapSubscriber
        tags:
            - { name: doctrine.event_subscriber }
        arguments:
            - "@sitemap.providers"
            - "@sitemap.generator"
```

```php
<?php

namespace JeroenDesloovere\SitemapBundle\EventListener;

use Doctrine\Common\EventSubscriber;
use Doctrine\Common\Persistence\Event\LifecycleEventArgs;
use Doctrine\ORM\Events;
use JeroenDesloovere\SitemapBundle\Generator\SitemapGenerator;
use JeroenDesloovere\SitemapBundle\Provider\SitemapProviders;

class SitemapSubscriber implements EventSubscriber
{
    /** @var SitemapGenerator */
    private $sitemapGenerator;

    /** @var SitemapProviders */
    private $sitemapProviders;

    public function __construct(SitemapProviders $sitemapProviders, SitemapGenerator $sitemapGenerator)
    {
        $this->sitemapProviders = $sitemapProviders;
        $this->sitemapGenerator = $sitemapGenerator;
    }

    public function getSubscribedEvents(): array
    {
        return [
            Events::postPersist,
            Events::postUpdate,
            Events::postRemove,
        ];
    }

    /**
     * @param LifecycleEventArgs $eventArgs
     * @throws \Exception
     */
    public function regenerate(LifecycleEventArgs $eventArgs): void
    {
        $entityClass = get_class($eventArgs->getObject());

        if ($this->sitemapProviders->exists($entityClass)) {
            $this->sitemapGenerator->regenerateForSitemapProvider($this->sitemapProviders->get($entityClass));
        }
    }

    public function postPersist(LifecycleEventArgs $eventArgs): void
    {
        $this->regenerate($eventArgs);
    }

    public function postRemove(LifecycleEventArgs $eventArgs): void
    {
        $this->regenerate($eventArgs);
    }

    public function postUpdate(LifecycleEventArgs $eventArgs): void
    {
        $this->regenerate($eventArgs);
    }
}
```

## Documentation

The class is well documented inline. If you use a decent IDE you'll see that each method is documented with PHPDoc.

## Contributing

Contributions are **welcome** and will be fully **credited**.

### Pull Requests

> To add or update code

- **Coding Syntax** - Please keep the code syntax consistent with the rest of the package.
- **Add unit tests!** - Your patch won't be accepted if it doesn't have tests.
- **Document any change in behavior** - Make sure the README and any other relevant documentation are kept up-to-date.
- **Consider our release cycle** - We try to follow [semver](http://semver.org/). Randomly breaking public APIs is not an option.
- **Create topic branches** - Don't ask us to pull from your master branch.
- **One pull request per feature** - If you want to do more than one thing, send multiple pull requests.
- **Send coherent history** - Make sure each individual commit in your pull request is meaningful. If you had to make multiple intermediate commits while developing, please squash them before submitting.

### Issues

> For bug reporting or code discussions.

More info on how to work with GitHub on help.github.com.

### Coding Syntax

We use [squizlabs/php_codesniffer](https://packagist.org/packages/squizlabs/php_codesniffer) to maintain the code standards.
Type the following to execute them:
```bash
# To view the code errors
vendor/bin/phpcs --standard=psr2 --extensions=php --warning-severity=0 --report=full "src"

# OR to fix the code errors
vendor/bin/phpcbf --standard=psr2 --extensions=php --warning-severity=0 --report=full "src"
```
> [Read documentation about the code standards](https://github.com/squizlabs/PHP_CodeSniffer/wiki)

### Unit Tests

We have build in tests, type the following to execute them:
```bash
vendor/bin/phpunit tests
```

## Credits

- [Jeroen Desloovere](https://github.com/jeroendesloovere)
- [All Contributors](https://github.com/jeroendesloovere/sitemap-bundle/contributors)

## License

The module is licensed under [MIT](./LICENSE.md). In short, this license allows you to do everything as long as the copyright statement stays present.
