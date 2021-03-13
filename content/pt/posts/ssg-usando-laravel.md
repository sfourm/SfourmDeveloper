---
title: "SSG usando Laravel"
url: ssg-usando-laravel
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

## O que é SSG (Static Site Generation)

É o processo de gerar páginas estáticas (.html) de sua aplicação.

Muitos frameworks oferecem essa ferramenta (Hugo, Next.Js, Nuxt.Js e muitos outros). Aqui vou mostrar que também podemos fazer SSG de sua aplicação Laravel que já está online, usando o próprio Laravel.

## Qual a utilidade disso?

Provavelmente sua aplicação está nesse momento processando o mesmo conteúdo para cada usuário que acessa sua homepage ou várias outras páginas do seu site, fazendo todo o processo de consumir dados no banco de dados ou cache, processá-los, gerar as views e renderizar um template para devolver ao usuário, de novo e de novo...

Após gerarmos páginas estáticas, esse processo acima não precisará mais acontecer a cada requisição em sua aplicação, pois após termos gerado um .html para cada página, nosso servidor nginx (ou qualquer outro) irá apenas retornar esse .html protinho para o usuário, sem a necessidade de nem enconstar na aplicação.

## Como fazer SSG usando Laravel?

Mostrarei abaixo duas maneiras, as duas podem ser usadas ao mesmo tempo (eu uso, inclusive), pois cada uma atende uma necessidade diferente. Para ambas as maneiras, utilizo uma configuração de disco em `config/filesystems.php` para definir para onde vão os .html gerados:

```php
'html' => [
    'driver'     => 'local',
    'root'       => public_path('cache-html'),
    'url'        => env('APP_URL'),
    'visibility' => 'public',
],
```

### 1. Geração automática em background job

Dessa maneira, um job irá realizar em background a geração das páginas estáticas que você predefinir.

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

E pronto, você pode criar um comando para disparar esse job a cada novo deploy da sua aplicação e também criar `events/observers/listeners` para gerar novamente os .html a cada alteração de conteúdo.

### 2. Geração após o término de uma requisição

Dessa maneira, ao término de uma requisição que contenha nosso middleware, o conteúdo devolvido ao usuário será escrito em um .html em nosso disco.

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

Registre esse middleware em `$routeMiddleware` no arquivo App\Http\Kernel`

```php

protected $routeMiddleware = [
    # ...
    'cache.html' => CacheHtmlResponse::class,
    #...
];
```

E adicione o middleware nas rotas que deseja páginas estáticas.

Por que também utilizar essa maneira, e não só definir todas as urls no background job mostrado anteriormente?

Imagine um blog com centenas ou milhares de posts, o job levaria vários minutos para terminar, e provavelmente, nem todos os blog posts recebem muitas requisições.

Esse é o cenário perfeito para usar esse middleware. Após o primeiro leitor receber a renderização do blog post, o .html daquela postagem será gerado e o próximo leitor não precisará esperar pelo processo de renderização novamente.

## Configurando Nginx para responder os .html gerados

Com a configuração abaixo, o nginx irá sempre procurar um .html para as urls que tem os prefixos definidos. Caso não encontre, irá seguir o fluxo normal e mandar a requisição para sua aplicação.

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

## Conclusão

Após os passos acima a aplicação finalmente fica responsável pelo que mais importa, como o checkout e dashboard do usuário. O nginx pode ficar com o processo mais leve e repetitivo de devolver páginas estáticas para o usuário. Além de o tempo de cada requisição diminuir bastante para essas páginas geradas estaticamente.
