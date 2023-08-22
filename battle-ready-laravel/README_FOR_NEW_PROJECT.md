# PROJECT

## Technologies

1) [Laravel](https://laravel.com/docs) - Laravel Framework
2) [Sail](https://laravel.com/docs/sail) - start services by docker
3) [Pint](https://laravel.com/docs/pint) - Code Style
4) [Larastan](https://github.com/nunomaduro/larastan) - PHPStan wrapper for Laravel
5) [Debugbar](https://github.com/barryvdh/laravel-debugbar) - Debugbar for Laravel
6) [Livewire](https://laravel-livewire.com/) - Make building dynamic interfaces simple for Laravel

## The git branching model in a project

Two branches type: `main`, `feature/`

1) From branch `main` create a new branch `feature/<id>-task-example`, where `<id>` - id task, `task-example` - name
   of task
2) Do task, add commits to branch `feature/19-task-example` and push branch to github
3) Next press button "Rebase and Merge" `rebase and merge`
4) After delete branch

## Run a project locally

For local starting we can use package [sail](https://laravel.com/docs/10.x/sail)

- It is necessary to install [docker](https://docs.docker.com/engine/install/)
- Optional install [composer 2.x](https://getcomposer.org/download/)
- *For windows need to work in [wsl](https://learn.microsoft.com/en-us/windows/wsl/install)*

1) Clone repo `git clone `
2) Go to directory repository and execute next commands (3-7)
3) a) If composer installed, then we can use next command: ```composer install```

   b) Otherwise install composer dependencies throw docker. The relevance of the command can be checked
   by [link](https://laravel.com/docs/sail#installing-composer-dependencies-for-existing-projects)
    ```
    docker run --rm \
    -u "$(id -u):$(id -g)" \
    -v "$(pwd):/var/www/html" \
    -w /var/www/html \
    laravelsail/php82-composer:latest \
    composer install --ignore-platform-reqs
    ```
4) ```cp .env.example .env```
5) ```./vendor/bin/sail up``` or ```./vendor/bin/sail up -d```
6) ```./vendor/bin/sail artisan key:generate```
7) ```./vendor/bin/sail artisan storage:link```
8) ```./vendor/bin/sail artisan migrate```
