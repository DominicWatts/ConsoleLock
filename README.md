# Magento 2 and Symfony lock composent

Experiment using Symfony lock compontent and magento console command. For locking console commands and cron proceses to prevent overlap, duplicate proceses or overloaded resource.

https://symfony.com/doc/current/components/lock.html#flockstore

## Install

    composer require symfony/lock

## Usage

### FlockStore

    https://symfony.com/doc/current/components/lock.html#lock-store-flock

### Example console command

```php
[...]
use Magento\Framework\Filesystem\DirectoryList;
use Symfony\Component\Lock\LockFactory;
use Symfony\Component\Lock\Store\FlockStore;
[...]
```

```php
/**
 * Console Test script
 * @param \Magento\Framework\Filesystem\DirectoryList $dir
 */
public function __construct(
    DirectoryList $dir
) {
    $this->dir = $dir;
    parent::__construct();
}
```

```php
$store = new FlockStore($this->dir->getPath('var'));
$factory = new LockFactory($store);
```

#### Example 1

```php
$lock = $factory->createLock('lock-name');
if ($lock->acquire()) {
    // do stuff
    $lock->release();
} else {
    // cannot get lock
}
```

#### Example 2

```php
$lock = $factory->createLock('lock-name', 30);
$lock->acquire();
try {
    // perform a job during less than 30 seconds
} catch (\Exception $e) {
    // whoops - error
} finally {
    $lock->release();
}
```

### Sample Console script

```php
<?php

declare(strict_types=1);

namespace Xigen\ConsoleLock\Console\Command;

use Magento\Framework\App\Area;
use Magento\Framework\App\State;
use Magento\Framework\Filesystem\DirectoryList;
use Symfony\Component\Console\Command\Command;
use Symfony\Component\Console\Input\InputInterface;
use Symfony\Component\Console\Output\OutputInterface;
use Symfony\Component\Lock\LockFactory;
use Symfony\Component\Lock\Store\FlockStore;

class Run extends Command
{
    /**
     * @var DirectoryList
     */
    protected $dir;

    /**
     * @var State
     */
    private $state;

     /**
      * Console Test script
      * @param \Magento\Framework\Filesystem\DirectoryList $dir
      * @param \Magento\Framework\App\State $state
      */
    public function __construct(
        DirectoryList $dir,
        State $state
    ) {
        $this->dir = $dir;
        $this->state = $state;
        parent::__construct();
    }

    /**
     * {@inheritdoc}
     */
    protected function execute(
        InputInterface $input,
        OutputInterface $output
    ) {
        $this->state->setAreaCode(Area::AREA_GLOBAL);
        $output->writeln('<info>Start</info>');

        $store = new FlockStore($this->dir->getPath('var'));
        $factory = new LockFactory($store);

        $output->writeln('<info>Creating lock</info>');
        $lock = $factory->createLock('test-console-run');
        if ($lock->acquire()) {
            $output->writeln('<warning>Releasing lock in 60 secs</warning>');
            // fake long running process
            sleep(60);
            $output->writeln('<info>Releasing lock</info>');
            $lock->release();
        } else {
            $output->writeln('<error>Cannot aquire lock</error>');
        }
        $output->writeln('<info>Finish</info>');
    }

    /**
     * {@inheritdoc}
     */
    protected function configure()
    {
        $this->setName("xigen:consolelock:run");
        $this->setDescription("Test lock");
        parent::configure();
    }
}
```