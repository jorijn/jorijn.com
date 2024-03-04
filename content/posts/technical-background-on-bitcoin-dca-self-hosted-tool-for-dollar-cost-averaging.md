---
title:
  "Technical background on Bitcoin DCA: self-hosted tooling for Dollar Cost
  Averaging with Bitcoin"
date: 2020-12-26T09:00:00
draft: false
url: /technical-background-on-bitcoin-dca-self-hosted-tool-for-dollar-cost-averaging/
tags:
  - bitcoin
---

{{< imgh src="bitcoin-dca-technical-post-header.jpeg" alt="Photo of a golden Bitcoin coin, on some money" >}}

At the beginning of 2020, I created an application to buy small amounts of
Bitcoin recurringly. This post is an effort to detail why, which problems arose,
and how I solved them.

<!--more-->

## An introduction

Every month I put aside a small portion of my income into Bitcoin. I used to do
this with an automated service called [Bittr](https://getbittr.com). Their goal
was to be easy to use. You would set up a recurring bank transfer, and when the
money came in, they would automatically convert it into Bitcoin and transfer it
to your wallet. When the Anti-Money Laundering bill (AMLD5) came into effect,
they, unfortunately, decided to shut down on matters of principle and hefty
registration fees.

While seeking a replacement, I quickly found there was none. I decided it would
be good to jump in the gap I created and put myself out by creating a
free-to-use open-source application.

## What goal was I trying to achieve

The tool I was designing would have to do the following things:

- Effortlessly buy Bitcoin in an unattended matter;
- Be able to communicate with multiple exchanges through their API;
- Be able to recurringly withdraw purchased Bitcoin from the exchange into a
  configured wallet;
- Be able to generate a new fresh wallet address for privacy purposes from a
  Master Public Key;
- It should run everywhere. Expecting people to install several PHP components
  and configure them properly is a hassle.

## Supporting multiple exchanges using Symfony service tags

My heart belongs to [BL3P](https://bl3p.eu). It's a simple, no-nonsense exchange
established in The Netherlands. At first, this was the only exchange Bitcoin DCA
supported. Quickly I realized it would be beneficial for the tool's future if I
would create a system where the project would support multiple exchanges. I
started designing interfaces to abstract behavior from the technical specifics
each platform required. I came up with these interfaces:

- BuyServiceInterface
- WithdrawServiceInterface
- BalanceServiceInterface
- WithdrawAddressProviderInterface

Each exchange implementation of the given interface gets marked using
[Symfony's service tags](https://symfony.com/doc/current/service_container.html).
When using Symfony's Dependency Injection container, you can configure it to get
all the tagged services and inject them as an argument into the configured
service. For example:

```yaml
service.buy.bitvavo:
  class: Jorijn\Bitcoin\Dca\Service\Bitvavo\BitvavoBuyService
  tags:
    - { name: exchange-buy-service }

service.buy.bl3p:
  class: Jorijn\Bitcoin\Dca\Service\Bl3p\Bl3pBuyService
  tags:
    - { name: exchange-buy-service }

service.buy.kraken:
  class: Jorijn\Bitcoin\Dca\Service\Kraken\KrakenBuyService
  tags:
    - { name: exchange-buy-service }
```

These are all the implementations of the
[BuyServiceInterface](https://github.com/Jorijn/bitcoin-dca/blob/master/src/Service/BuyServiceInterface.php)
available. The coordinating service for issuing buy orders gets them injected
like this:

```yaml
service.buy:
  class: Jorijn\Bitcoin\Dca\Service\BuyService
  arguments:
    - !tagged_iterator exchange-buy-service
    - "%env(EXCHANGE)%"
```

Eventually, all instantiated objects get injected into the BuyService using its
constructor. The code doesn't need to know about available exchanges and which
one the user prefers. It's all nicely abstracted away into the configuration.
The result is:

```php
class BuyService
{
    /** @var BuyServiceInterface[]|iterable */
    protected $registeredServices;
    protected string $configuredExchange;

    public function __construct(iterable $registeredServices, string $configuredExchange) {
        $this->registeredServices = $registeredServices;
        $this->configuredExchange = $configuredExchange;
    }

    public function buy(int $amount, string $tag = null): CompletedBuyOrder
    {
        foreach ($this->registeredServices as $registeredService) {
            if ($registeredService->supportsExchange($this->configuredExchange)) {
                return $registeredService->initiateBuy($amount);
            }
        }

        throw new NoExchangeAvailableException('some descriptive error here');
    }
}
```

## Generating a new wallet address for each withdrawal

In Bitcoin, it's considered good privacy to use a new address for each
transaction. The blockchain is public, and reusing earlier used addresses could
tell others your balance and behaviors. Most modern software provides
Hierarchical Deterministic (HD) wallets. In short, one single key can generate a
hierarchical tree-like structure of multiple private/public keys. My goal was to
leverage this so every time a user would request a withdrawal, the tool would
provide a new, unused address.

A new problem arose, how would the tool know if an address was previously used
or not? This information is available since Bitcoin saves every transaction in
the public blockchain. The need to know poses a few concerns:

- Infrastructural: You would need a blockchain index to query. There are two
  options available:
  - I would have to provide a hosted service to figure out if transactions exist
    on the given address.
  - The user would have to host such a service themselves.
- Psychological: I don't want to be able to know if an address is active
  already. The Bitcoin community is rigorous in who they trust. Preferably no
  one.

The infrastructural argument increases complexity enormously. Bittr solved this
problem by hosting the service themselves, but this wasn't a perfect option
since I want the tool to operate without any dependencies linking back to me.

There was another option. Admittedly, it's not very pretty and prone to error,
but it provides trustless and functional requirements. I would document my
recommendation to create a new wallet for the sole purpose of using my software.
I could reasonably assume the first address is unused and increment an internal
counter whenever the user requests a withdrawal.

## Thinking about how to run the application on a bunch of different configurations

If you have ever developed any PHP application, you know this isn't pretty. The
upside is that PHP is readily available on the system most of the time or that
it's easy enough to arrange so. The downside is you don't know the person
maintaining the system, how often they upgrade, and which upstream repositories
they use. PHP 7.4 was stable when writing the application, so I wanted to use
typed properties, arrow functions, spread operators, etcetera. I couldn't
possibly assume everyone would be up to date.

I chose Docker: Most modern operating systems run it, and Docker gives me a
predictable set of libraries and dependencies to launch my application. It's
portable and allows for multi-architectural builds. The minimal viable product
(MVP) doesn't require a graphical user interface, so the plan was to package it
as a CLI application.

There's one caveat: Containers are ephemeral—lack of persistency and state
management force the developer to think about the application's life-cycle and
how one should feed configuration and process results.

Symfony provides good support for consuming configuration through environment
variables and suits nicely in the context of containers. For more advanced
functionality that does require persistency, I could document a volume mount for
storage.

Using Docker's multi-stage builds, I can also consistently prove every part of
the application works before shipping. My
[Dockerfile](https://github.com/Jorijn/bitcoin-dca/blob/master/Dockerfile)
contains four stages:

- **The dependency stage** is a preparatory layer for the final image. Here, all
  dependencies get installed using Composer. Ultimately, this layer gets thrown
  away since we only need the dependency folder, not the entire layer.
- **The base stage** is where the previous step's dependencies are copied over
  and where the required PHP extensions are installed. For ease, this stage
  builds upon the existing "php:7.4-cli-alpine" image.
- **The test stage** builds upon the base stage. It copies development settings
  for PHP and runs all available tests. The goal is to ensure the build doesn't
  succeed when a part of the application shows unintended behavior.
- **The production stage** is based upon the base stage and purposely skips the
  added development files added in the test stage. Docker will only reach this
  stage if all tests succeed. Here, production config & entry points get added,
  and the container is pre-compiled for performance reasons.

The final result is a tested image that behaves consistently for everyone:

```shell
$ docker run --rm -it \
    -e SOME_CONFIG1=value1 \
    -e SOME_CONFIG2=value2 \
    jorijn/bitcoin-dca:latest buy 10
```

## The Raspberry Pi problem

A little while after I launched the tool publicly,
[some](https://github.com/Jorijn/bitcoin-dca/issues/42)
[issues](https://github.com/Jorijn/bitcoin-dca/issues/31) came in. As it turned
out, Raspberry Pi computers are excellent little devices to run Bitcoin DCA on:
They run Linux, are small, and most important: Available 24/7 for a recurring
schedule. The caveat: most Raspberry Pi's run ARMv7 architecture, which is
32-bit.

**The first obstacle**: The library I was using for
[Key Derivation](https://github.com/Bit-Wasp/bitcoin-php) was unable to work
since 32-bit PHP cannot handle large integers. On 64-bit PHP, the library allows
me to supply a Master Public Key and provide it with an index integer. It will
return the corresponding private/public keypairs to process the address for
withdrawal.

32-bit PHP unable to handle big integers to derive private/public keys turned
out to be a tricky problem. I spent several nights trying suggestions and
searching on Google to find a cure for this bug. Ultimately, I didn't find one.
I soon learned that Python has many libraries for working with Bitcoin that run
on 32-bit systems. In collaboration with my friend, we quickly whipped up a
simple script that accepts a Master Public Key, offset, and length parameter and
returns a list of recipient addresses.

**The second obstacle**: How would I connect the two? I would need some bridge
connecting the two APIs for exchanging information. HTTP or TCP wasn't suitable
here since the ephemeral nature of containers. Starting a daemon to handle API
traffic and spawning another process to talk from seemed daunting and
complicated. I settled on another CLI script: Thanks to Docker, I know the
filesystem structure, where the Python tool would be, and which dependencies are
available.

**The third obstacle**: How would I provide a graceful fallback during runtime
when 32-bit PHP is detected? I decided on leveraging Symfony's service tag's
again here. The interface I designed looks like this:

```php
interface AddressFromMasterPublicKeyComponentInterface
{
    public function derive(string $masterPublicKey, $path = '0/0'): string;

    public function supported(): bool;
}
```

It consists of two methods: `derive` and `supported`. One great thing about
service tags is that you can prioritize them. I created two implementations of
this interface. 1) the solution native to the application and 2) falling back to
executing the Python CLI tool on the system level.

```php
component.derive_from_master_public_key_bitwasp:
  class: Jorijn\Bitcoin\Dca\Component\AddressFromMasterPublicKeyComponent
  tags:
    - { name: derive-from-master-public-key, priority: -500 }

component.derive_from_master_public_key_external:
  class: Jorijn\Bitcoin\Dca\Component\ExternalAddressFromMasterPublicKeyComponent
  tags:
    - { name: derive-from-master-public-key, priority: -1000 }
```

The native solution prioritizes first. It checks if `PHP_INT_SIZE` it equals 8,
meaning the system is 64-bit capable. If not, the application proceeds to the
next available PublicKeyComponent.

```php
public function supported(): bool
{
    // this component only works on PHP 64-bits
    return \PHP_INT_SIZE === 8;
}
```

The Python solution comes last since it's always supported. I decided against
solely using the Python solution since it was very slow compared to the native
PHP solution. Using a Factory pattern injected with all tagged
PublicKeyComponents, I could return the interface implementation while the
requesting service doesn't need to know the specifics:

```php
public function createDerivationComponent(): AddressFromMasterPublicKeyComponentInterface
{
    foreach ($this->availableComponents as $availableComponent) {
        if (true === $availableComponent->supported()) {
            return $availableComponent;
        }
    }

    throw new NoDerivationComponentAvailableException('no derivation component is available');
}
```

The DI container nicely handles the logic for setting up the service using the
factory:

```yaml
component.derive_from_master_public_key:
  class: Jorijn\Bitcoin\Dca\Component\AddressFromMasterPublicKeyComponentInterface
  factory:
    [
      "@factory.derive_from_master_public_key.component",
      "createDerivationComponent",
    ]
```

## Why code quality and tests matter

I get it; writing tests is tedious work. It's not fun, and developers don't like
it. Most often, writing a good unit test that covers all logical paths in the
code could take up to 100-200% of the time initially needed for the component
itself.

Tests make it easier for us to refactor code. Over time projects change. Unit
tests verify that updates to code won't break any existing functionality that
has previously been thoroughly tested. We can refactor code and run our tests to
ensure that it still works as originally intended.

Most developers know that colleague that continually plagues them about
maintaining consistent code quality. We grunt, sigh, and secretly know they are
right. I solely support Bitcoin DCA, and unfortunately, I don't have that
colleague. I recruited a replacement in an automated online system called
SonarQube, which provides gatekeeper functionality for quality. It continuously
inspects my code and warns me about issues in maintainability, reliability, and
security. The best thing?
[It's verifiable for everyone to see](https://sonarcloud.io/dashboard?id=Jorijn_bitcoin-dca).

When writing this article, 92.3% of Bitcoin DCA's codebase is
[covered by tests](https://sonarcloud.io/component_measures?id=Jorijn_bitcoin-dca&metric=coverage&view=list),
and there are little to no issues regarding
[maintainability / technical debt](https://sonarcloud.io/component_measures?id=Jorijn_bitcoin-dca&metric=Maintainability&view=list).
I'm proud of this project as it's a genuine representation of how I think code
quality and test coverage should be.

## Conclusion

I learned a lot by creating Bitcoin DCA. I never knew 32-bit architecture was
still active in 2020. The Dutch Bitcoin community has been great in supporting
me. Not long after the initial launch, I even got a few mentions on
[Dutch news media](https://bitcoinmagazine.nl/bitcoin-nieuws-overzicht/bitcoin-dollar-cost-averaging-tool-jorijn).

## What's next

In the next couple of months, I'm looking to complete/implement these new
features:

- Notifications of completed orders through email or instant messaging.
- Kraken recently announced they would be supporting withdrawals through
  Bitcoin's Lightning network somewhere in 2021. I think this is a great use
  case for Bitcoin DCA.

## Give Bitcoin (DCA) a try

I wrote [detailed documentation](https://bitcoin-dca.readthedocs.io/en/latest/)
on how to use Bitcoin DCA yourself. Visit the repository and download/inspect
the code
[here](https://web.archive.org/web/20230929024452/https://github.com/Jorijn/bitcoin-dca)
