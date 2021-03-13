---
title: "SSG using Laravel"
url: ssg-using-laravel
date: 2021-03-07T09:33:16-03:00
draft: true
tags: ["laravel"]
categories: ["tutorials"]
imgs:
  [
    "../ssg-static-site-generation.png",
  ]
ogimage: "https://ibrunotome.github.io/ssg-static-site-generation.png"
comments: true
draft: false
translationKey: "ssg-using-laravel"
---

## What is SSG (Static Site Generation)

It is the process of generating static pages (.html) of your application.

Many frameworks offer this tool (Hugo, Next.Js, Nuxt.Js and many others). Here I will show that we can also make SSG of your Laravel application that is already online, using Laravel itself.

## What is the use of this?

Probably your application is currently processing the same content for each user who accesses your homepage or several other pages on your site, doing the entire process of consuming data in the database or cache, processing them, generating the views and rendering a template to return to the user, again and again...

After generating static pages, the above process will no longer need to happen to each request in your application, because after we have generated an .html for each page, our nginx server (or any other) will just return this .html to the user, without the need to hit the application.

## How to do SSG using Laravel?

I will show you two methods below, both can be used at the same time, as each meets a different need. For both methods, I use a disk configuration in `config/filesystems.php` to define where the generated .html files go:

```php
'html' => [
    'driver'     => 'local',
    'root'       => public_path('cache-html'),
    'url'        => env('APP_URL'),
    'visibility' => 'public',
],
```

### 1. Automatic generation in background job

In this way, a job will perform in the background the generation of static pages that you preset.

```php
<?php

namespace App\Jobs;

use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;
use Illuminate\Support\Facades\Cache;
use Illuminate\Support\Facades\Http;
use Illuminate\Support\Facades\Storage;

class GenerateStaticSiteJob implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public $timeout = 3600;

    public $tries = 1;

    private string $baseUrl = '<seu-dominio-aqui>';

    public function handle()
    {
        Cache::lock('ssg')->get(function () {
            $this->generateInitialPages();

            $urls = array_merge(
                $this->portugueseUrls(),
                $this->englishUrls(),
            );

            foreach ($urls as $url) {
                Storage::disk('html')->delete("{$url}.html");
                $content = Http::get("{$this->baseUrl}/{$url}")->body();
                Storage::disk('html')->put("{$url}.html", $content);
            }
        });
    }

    private function generateInitialPages(): void
    {
        Storage::disk('html')->delete('index.html');
        $content = Http::get("{$this->baseUrl}")->body();
        Storage::disk('html')->put('index.html', $content);

        Storage::disk('html')->delete('en/index.html');
        $content = Http::get("{$this->baseUrl}/en?v={$version}")->body();
        Storage::disk('html')->put('en/index.html', $content);

        Storage::disk('html')->delete('blog/index.html');
        $content = Http::get("{$this->baseUrl}/blog")->body();
        Storage::disk('html')->put('blog/index.html', $content);
    }

    private function portugueseUrls(): array
    {
        return [
            'servicos',
            'avaliacoes-de-clientes',
            'termos-de-servico',
            'sobre-nos',
            'politica-de-privacidade',
            'politica-de-cancelamento',
        ];
    }

    private function englishUrls(): array
    {
        return [
            'en/services',
            'en/customer-reviews',
            'en/terms-of-service',
            'en/about-us',
            'en/privacy-policy',
            'en/cancel-policy',
        ];
    }
}
```

Thats it! You can create a command to trigger this job with each new deployment of your application and also create `events/observers/listeners` to generate the .html again with each content change.

### 2. Generation after the end of a requisition

Thus, at the end of a request that contains our middleware, the content returned to the user will be written in an .html on our disk.

```php
<?php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Illuminate\Http\Response;
use Illuminate\Support\Facades\Storage;

class CacheHtmlResponse
{
    public function handle($request, Closure $next)
    {
        return $next($request);
    }

    public function terminate(Request $request, $response)
    {
        if (!app()->environment('production')) {
            return;
        }

        if (!$response instanceof Response) {
            return;
        }

        if (!is_null($request->getQueryString())) {
            return;
        }

        if ($response->getStatusCode() !== Response::HTTP_OK) {
            return;
        }

        $content = $response->getContent();
        $pathParts = explode('/', trim($request->getPathInfo(), '/'));
        $filePart = array_pop($pathParts);
        $file = (strlen($filePart) ? $filePart : "index") . '.html';
        $relativePath = implode("/", $pathParts);

        Storage::disk('html')->put(
            $relativePath . "/" . $file,
            $content,
            ['CacheControl' => 'public,max-age=60,no-transform']
        );
    }
}
```

Register this middleware in `$routeMiddleware` in the file App\Http\Kernel`

```php
protected $ routeMiddleware = [
     # ...
     'cache.html' => CacheHtmlResponse :: class,
     # ...
];
```

And add middleware on the routes you want to static pages.

Why also use this way instead of just define all the urls in the background job shown above?

Imagine a blog with hundreds or thousands of posts, the job would take several minutes to finish, and probably not all blog posts receive many requests.

This is the perfect scenario for using this middleware. After the first reader receives the rendering of the blog post, the .html for that post will be generated and the next reader will not have to wait for the rendering process again.

## Configuring Nginx to answer the generated .html

With the configuration below, nginx will always look for a .html for urls that have prefixes defined. If not, it will follow the normal flow and send the request to your application.

```nginx
location ~ ^/(servicos|blog|avaliacoes-de-clientes|termos-de-servico|sobre-nos|politica-de-privacidade|politica-de-cancelamento).*$ {
  try_files /cache-html/$uri.html$arg_page $uri $uri/ /index.php?$args;
}

location ~ ^/en\/?(services|customer-reviews|terms-of-service|about-us|privacy-policy|cancel-policy).*$ {
  try_files /cache-html/$uri.html$arg_page $uri $uri/ /index.php?$args;
}

location / {
  try_files $uri $uri/ /index.php?$query_string;
}

location ~ \.php$ {
  try_files $uri =404;
  fastcgi_split_path_info ^(.+\.php)(/.+)$;
  fastcgi_pass app:9000;
  fastcgi_index index.php;
  include fastcgi_params;
  fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
  fastcgi_param PATH_INFO $fastcgi_path_info;
  fastcgi_read_timeout 180;
  proxy_set_header Host            $http_host;
  proxy_set_header X-Real-IP       $remote_addr;
  proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
}
```

## Conclusion

After the steps above, the application is finally responsible for what matters, such as the user's checkout and dashboard. Nginx can have the lightest and most repetitive process of returning static pages to the user. In addition, the time of each request decreases considerably for these statically generated pages.