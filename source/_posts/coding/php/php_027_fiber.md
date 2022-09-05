---
title: 'php[027]fiber'
date: '2022-09-05T20:40:00+08:00'
tags:
    - php
categories:
    - other

---

> 大家的电脑应该都是大等于2核的了，但是大家电脑上同时运行的程序大多远远多于cpu的核心数量。这是因为操作系统在任务处理上采取了宏观上并行，微观上串行的做法。也就是cpu每个程序都执行了一点点时间然后就切换去执行别的程序。使得大家看上去都执行了很多。现在 php8.1 。推出了 fiber 。把调度权利赋予给了各位 php 开发。那么我们有 fiber 我们可以实现什么样的新操作呢。

拿平时大家写的 for 循环举例。像 go 你可以写两个 `go` 每个里面各写一个循环同时输入，你可以看到输出是交替。在过去的php版本中，如果只开启一个 cli 写多个 for 循环，那么他的输出一定是顺序的。无法做到交叉输出（也就是无法在第一个循环中执行若干次后，让b再执行，b执行一段时间后，再让A执行）。现在借助 fiber 我们也可以实现这种操作。下面这段代码就可以做到两个循环交叉执行。甚至可以控制两个程序执行的频率（比如A执行3次，B执行一次这样分配）

<!--more-->


```php
<?php
$t1    = false;
$t2    = false;
$reg   = [];
$reg[] = new \Fiber(function () use (&$t1) {
    for ($i = 1; $i < 10; $i++) {
        echo $i;
        echo PHP_EOL;
        \Fiber::suspend();

    }
    $t1 = true;
});


$reg[] = new \Fiber(function () use (&$t2) {
    for ($i = 1; $i < 10; $i++) {
        echo $i;
        echo PHP_EOL;
        \Fiber::suspend();
    }
    $t2 = true;
});

$startTag = true;
while (count($reg) > 1) {

    if ($startTag) foreach ($reg as $pI) {
        $pI->start();
        $startTag = false;
    }

    foreach ($reg as $pI) {
        $pI->resume();
    }

    if ($t1 === true && $t2 === true) {
        break;
    }
}
```
```
1
1
2
2
3
3
4
4
5
5
6
6
7
7
8
8
9
9
```

你甚至可以控制两个循环的执行频率，比如 第一个循环 执行3次后，第二个循环执行一次。代码如下


```php
<?php
$reg = [];
$fId = 1;


$reg[$fId] = new \Fiber(function () use (&$reg, $fId) {
    for ($i = 1; $i < 10; $i++) {
        echo $fId . ':' . $i;
        echo PHP_EOL;
        if ($i % 3 == 0) {
            \Fiber::suspend();
        }
    }
    unset($reg[$fId]);
});
$fId++;

$reg[$fId] = new \Fiber(function () use (&$reg, $fId) {
    for ($i = 1; $i < 10; $i++) {
        echo $fId . ':' . $i;
        echo PHP_EOL;
        \Fiber::suspend();
    }
    unset($reg[$fId]);
});

$startTag = true;
while (count($reg) > 0) {
    if ($startTag) foreach ($reg as $pI) {
        $pI->start();
        $startTag = false;
    }
    foreach ($reg as $pI) {
        $pI->resume();
    }
}

```

```
1:1
1:2
1:3
2:1
1:4
1:5
1:6
2:2
1:7
1:8
1:9
2:3
2:4
2:5
2:6
2:7
2:8
2:9
```
通过消息通知完成

```php
<?php

namespace App\Command;

use Symfony\Component\Console\Attribute\AsCommand;
use Symfony\Component\Console\Command\Command;
use Symfony\Component\Console\Input\InputArgument;
use Symfony\Component\Console\Input\InputInterface;
use Symfony\Component\Console\Input\InputOption;
use Symfony\Component\Console\Output\OutputInterface;
use Symfony\Component\Console\Style\SymfonyStyle;

#[AsCommand(
    name: 'Sname',
    description: 'Add a short description for your command',
)]
class SnameCommand extends Command
{
    protected function configure(): void
    {
        $this
            ->addArgument('arg1', InputArgument::OPTIONAL, 'Argument description')
            ->addOption('option1', null, InputOption::VALUE_NONE, 'Option description');
    }

    protected function execute(InputInterface $input, OutputInterface $output): int
    {
        $t1  = false;
        $t2  = false;
        $reg = [];
        $fId = 1;


        $reg[] = new \Fiber(function () use ($fId) {
            for ($i = 1; $i < 10; $i++) {
                echo $fId . ':' . $i;
                echo PHP_EOL;
                if ($i % 3 == 0) {
                    \Fiber::suspend(new SuspendData(Status::Running));
                }
            }
            \Fiber::suspend(new SuspendData(Status::Stop));

        });
        $fId++;

        $reg[] = new \Fiber(function () use ($fId) {
            for ($i = 1; $i < 10; $i++) {
                echo $fId . ':' . $i;
                echo PHP_EOL;
                \Fiber::suspend(new SuspendData(Status::Running));
            }
            \Fiber::suspend(new SuspendData(Status::Stop));
        });

        $startTag = true;
        while (count($reg) > 0) {
            if ($startTag) foreach ($reg as $pI) {
                $pI->start();
                $startTag = false;
            }
            foreach ($reg as $key => $pI) {
                $r = $pI->resume();
                if ($r->status === Status::Stop) {
                  unset($reg[$key]);
                }
            }
        }


        return Command::SUCCESS;
    }
}

class SuspendData
{
    public readonly Status $status;

    public function __construct($status)
    {
        $this->status = $status;
    }
}

enum Status
{
    case Stop;
    case Running;
}
```

欢迎大家补充