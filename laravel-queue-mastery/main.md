# Laravel queue mastery

## Dispatching and Running Jobs
### Create Job
```shell
php artisan make:job ExampleJob
```
### handle a job2
```php
(new \App\Jobs\ExampleJob)->handle();
\App\Jobs\ExampleJob::dispatch();
```
### Store queue to database
```dotenv
QUEUE_CONNECTION=database
```
```shell
php artisan queue:table
php artisan migrate
```
### Start queue work
```shell
php artisan queue:work
```

## Configuring Jobs
### Delay Job
```php
\App\Jobs\ExampleJob::dispatch()->delay(5);
```
### Execution timeout of the job
```php
public $timeout = 1;
```
### Tries job if it fails
```php
public $tries = 3;
```
### Retry until job if it fails
```php
public $tries = -1;
public $backoff = 2; // delay in seconds between retries
public function retryUntil()
{
    return now()->addMinute();
}
```
### Start queue work in different terminals, help to asynchronous execute queue jobs
```shell
php artisan queue:work
php artisan queue:work
php artisan queue:work
```
### Queue separation
```php
\App\Jobs\PaymentsJob::dispatch()->onQueue('payments');
```
```shell
php artisan queue:work --queue=payments,default
```

## Handling Attempts & Failures
### Delay Job
```php
// Class properties \App\Jobs\ExampleJob.php
public $tries = 10;
public $backoff = [2, 10, 20]; // the delay after the first failed attempt will be 2, after the second 10, all others 20
```
### Retry failed job
```shell
php artisan queue:retry ce7bb17c-cdd8-41f0-a8ec-7b4fef4e5ece
```
### Release a job back onto the queue
```php
// Class properties and methods \App\Jobs\ExampleJob.php
public $tries = 3;
public function handle()
{
    return $this->release(2); // back job to queue after 2 seconds
}
```
### Maximum throw exceptions in job
```php
public $maxExceptions = 2; // class property \App\Jobs\ExampleJob.php
```
### Write log if job failed
```php
// Class properties and methods \App\Jobs\ExampleJob.php
public $tries = 10;
public $maxExceptions = 2;
public function handle()
{
    throw new \Exception();
    return $this->release();
}
public function failed($e) //this method execute when job will fail
{
    Illuminate\Support\Facades\Log::info('Failed');
}
```

## Dispatching Workflows
### Job chain
```php
// Job chaining allows you to specify a list of queued jobs that should be run
// If RunTests fail Deploy will not dispatch
\Illuminate\Support\Facades\Bus::chain([
    new \App\Jobs\PullRepo(),
    new \App\Jobs\RunTests(),
    new \App\Jobs\Deploy()
])->dispatch();
```
### Job batch
* Create a database migration
```shell
php artisan queue:batches-table
php artisan migrate
```
* Add trait to Job Class
```php
use Illuminate\Bus\Batchable\Batchable
```
* You can add method to Job Class if the batch has been cancelled
```php
public function handle()
{
    if ($this->batch()->cancelled()) {
        return;
    }
}
```
* Use code to execute a batch of jobs
```php
Bus::batch([
    new PullRepo('laracasts/project1'),
    new PullRepo('laracasts/project2'),
    new PullRepo('laracasts/project3')
])->dispatch();
```

## More Complex Workflows
### Job chain
```php
Bus::batch([
    new PullRepo('laracasts/project1'),
    new PullRepo('laracasts/project2'),
    new PullRepo('laracasts/project3')
])
    ->allowFailures() // a job failure does not automatically mark the batch as cancelled
    ->catch(function (Batch $batch, Throwable $e) {
        // first batch job failure detected...
    })->then(function (Batch $batch) {
        // all jobs completed successfully...
    })->finally(function (Batch $batch) {
        // the batch has finished executing...
    })
    ->onQueue('deployment') // specify the queue
    ->onConnection('database') // specify the connection
    ->dispatch();
```
### Execute both chains
```php
$batch = [
    [
        new PullRepo('laracasts/project1'),
        new RunTests('laracasts/project1'),
        new Deploy('laracasts/project1')
    ],
    [
        new PullRepo('laracasts/project2'),
        new RunTests('laracasts/project2'),
        new Deploy('laracasts/project2')
    ]
];
Bus::batch($batch)
    ->allowFailures()
    ->dispatch();
```
### Chain and chain closures with batch
```php
\Illuminate\Support\Facades\Bus::chain([
    new \App\Jobs\PullRepo(),
    function () {
        \Illuminate\Support\Facades\Bus::batch([...])->dispatch()
    }
])->dispatch();
```

## Controlling and Limiting Jobs
### Lock Job to prevent parallel execution with Cache
```php
// Class methods \App\Jobs\Example.php
public function handle()
{
    Cahce::lock('deployments')->block(10, function () {
        Log::info('Started Deploying...');
        sleep(5);
        Log::info('Finished Deploying...');
    });
}
```

### Job Middleware with redis
```php
// Class methods \App\Jobs\Example.php
public function handle()
{
    Redis::throttle('key')
        ->allow(10)
        ->every(60)
        ->block(10)
        ->then(function () {
            Log::info('Started Deploying...');
            sleep(5);
            Log::info('Finished Deploying...');
        });
}
```

### Preventing Job Overlaps
```php
// Class methods \App\Jobs\Example.php
public function handle()
{
    Log::info('Started Deploying...');
    sleep(5);
    Log::info('Finished Deploying...');
}
public function middleware()
{
    return [new WithoutOverlapping('deployments', 10)];
}
```

## More Job Configurations
### Unique Job
```php
// Class \App\Jobs\Example.php
implements Illuminate\Contracts\Queue\ShouldBeUnique
```
### Unique Job Until Processing
```php
// Class \App\Jobs\Example.php
implements Illuminate\Contracts\Queue\ShouldBeUniqueUntilProcessing
```
### Unique id and unique for
```php
// Class \App\Jobs\Example.php
public function uniqueId()
{
    return 'deployments';
}
public function uniqueFor()
{
    return 60;
}
```
### Throttle exceptions
```php
// Class \App\Jobs\Example.php
public function middleware()
{
    return [new ThrottlesExceptions(10)];
}
```


## Designing Reliable Jobs
### Unique Job
```php
// Class \App\Jobs\Example.php
implements Illuminate\Contracts\Queue\ShouldBeUnique
```
