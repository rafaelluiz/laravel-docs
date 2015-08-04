# Facades

- [Introdução](#introduction)
- [Explicação](#explanation)
- [Uso Prático](#practical-usage)
- [Criando Fachadas](#creating-facades)
- [Imitando Fachadas (Mocking)](#mocking-facades)
- [Referências de Calsse Fachada](#facade-class-reference)

<a name="introduction"></a>
## Introdução

Fachadas fornecem uma interface "estática" para classes que estão disponíveis no [container de serviço](/docs/{{version}}/container) das aplicações. Laravel já vem com várias fachadas, e você provavelmente você tem as usado sem ao menos conhece-las! "Fachadas" do Laravel servem como "proxies" para classes subjacentes no container de serviços, fornecendo o benefício de uma sintaxe concisa e expressiva enquanto é mantida mais testável e flexível do que os métodos estáticos tradicionais.

Facades provide a "static" interface to classes that are available in the application's [service container](/docs/{{version}}/container). Laravel ships with many facades, and you have probably been using them without even knowing it! Laravel "facades" serve as "static proxies" to underlying classes in the service container, providing the benefit of a terse, expressive syntax while maintaining more testability and flexibility than traditional static methods.

Ocasionalmente, você pode desejar criar sua própria fachada para sua aplicação e pacotes, então vamos explorar o conceito de desenvolvimento e uso dessas classes.

> **Nota:** Antes de mergulhar no mundo das fachda, é fortemente recomendado que você se familiarize com o [container de serviços](/docs/{{version}}/container) do Laravel. 

<a name="explanation"></a>
## Explicação

No contexto da aplicação Laravel, uma fachada é uma classe que forncece acesso para um objeto de um container. Toda a engenharia que faz este trabalho está na classe `Facade`. As fachadas do Laravel, e qualquer outra fachada customizada que você criar, irá extender da classe base `Facade`.

Sua classe fachada apenas necessita implementar um único método `getFacadeAccessor`. É trabalho do método `getFacadeAccessor` definir o que resolver a partir do container. A classe base `Facade` fa uso do método-mágico `__callStatic()` para adiar as chamadas da sua classe fachada para resolver o objeto.

Então, quando você faz a chamada de uma fachada como por exemplo: `Cache::get`, o Laravel resolve a classe gerenciadora Cache de fora do container de serviços e chama o método `get` da classe. Em termos técnicos, Fachadas Laravel são uma sintaxe conveniente para usar o container de serviços Laravel como um localizador de serviços.

<a name="practical-usage"></a>
## Uso Prático

No exemplo abaixo, a chamada é feita para o sistema de cache do Laravel. Ao olhar para este código, pode-se supor que o método estático  `get` é chamado na clsse `Cache`.
In the example below, a call is made to the Laravel cache system. By glancing at this code, one might assume that the static method `get` is being called on the `Cache` class.

	$value = Cache::get('key');

Contudo, se nos olharmos para classe `Illuminate\Support\Facades\Cache`, você verá que lá não existe um método estático `get`:

	class Cache extends Facade {

		/**
		 * Get the registered name of the component.
		 *
		 * @return string
		 */
		protected static function getFacadeAccessor() { return 'cache'; }

	}

A classe Cache extende da classe base `Facade` e define o método  `getFacadeAccessor()`. Lembre-se, o objetivo deste método é retornar o nome da ligação do container de serviços.

Quando um usuário faz referência a qualquer método estático na fachada `Cache`, Laravel resolve a ligação `cache` a partir do container de serviços e executa o método requisitado (neste caso, `get`) contra esse objeto.

Então, nossa chamada `Cache::get` pode ser re-escrita assim:

	$value = $app->make('cache')->get('key');

#### Importando Fachadas 

Lembre-se, se você está usando uma fachada em um controlador que faz um de namespace, você precisará importar a classe da fachada no namespace. Todas as fachadas estão localizadas no namespace global:

	<?php namespace App\Http\Controllers;

	use Cache;

	class PhotosController extends Controller {

		/**
		 * Get all of the application photos.
		 *
		 * @return Response
		 */
		public function index()
		{
			$photos = Cache::get('photos');

			//
		}

	}

<a name="creating-facades"></a>
## Criando Fachadas

Criar fachadas para sua própria aplicação ou pacote é simples. Você apenas precisa de três coisas:

- Uma ligação de container de serviços
- A classe Fachada.
- Uma configuração "alias" da fachada

Vamos dar uma olhada neste exemplo. Aqui, nos temos a classe definada como `PaymentGateway\Payment`.

	namespace PaymentGateway;

	class Payment {

		public function process()
		{
			//
		}

	}
Nos precisamos ser capazes de resolver esta classe a partir do container de serviços. Então, vamos adicionar a aligação para um fornecedor de serviços:

	App::bind('payment', function()
	{
		return new \PaymentGateway\Payment;
	});

Um ótimo lugar para registrar esta ligação seria criando um novo [fornecerdor de serviços](/docs/{{version}}/container#service-providers) chamado `PaymentServiceProvider`, e adicionar esta ligação ao método `register`. Você pode então configurar para que o Laravel carregue seu fornecedor de serviços a partir do arquivo de configuração `config/app.php`.

Após isto, nos podemos criar nosa própria classe fachada:


	use Illuminate\Support\Facades\Facade;

	class Payment extends Facade {

		protected static function getFacadeAccessor() { return 'payment'; }

	}

Finalmente, se quisermos, nos podemos adicionar um alias(apelido) para nossa fachada ao array `aliases` no arquivos de configuração `config/app.php`. Agora, nos podemos chamar o método `process` na instância da classe `Payment`. 

	Payment::process();

### Uma nota sobre auto-loading Aliases (Apelidos auto-carregáveis)

Classes no array `aliases` não estão disponíveis em algumas instâncias por que [PHP não irá tentar carregar automaticamente classes dom tipagens indefinidas](https://bugs.php.net/bug.php?id=39003). Se `\ServiceWrapper\ApiTimeoutException` é apelidado de `ApiTimeoutException`, a exeção `catch(ApiTimeoutException $e)` fora do namespace `\ServiceWrapper` nunca irá entrar na exeção, mesmo se uma for levantada. Um problema similar é encontrado em classes que tem tipagem para classes apelidadas (aliases). A única solução é não usar (aliasing, não apelidar classes) e `usar` as classes que você deseja tipando-as no começo de cada arquivo que as requisita.

<a name="mocking-facades"></a>
## Imitando Fachadas (Mocking)

Teste unitário é importante aspecto do porque fachadas funcionam do jeito que são. Na verdade, testabilidade é a razão primária para fahadas sequer existam. Para mais informações, dê uma olhada na seção de documentação [imitando fachadas](/docs/testing#mocking-facades) 

<a name="facade-class-reference"></a>
## Referências de Classe Fachada

Abaixo você irá encontrar todas fachadas e suas classes subjacentes. Isto é uma ferramenta útil para rapidamente cavar mais fundo na documentação da API para uma determinada raiz da fachada. A chave [ligação do container de serviços](/docs/{{version}}/container) também é incluída aonde é aplicável.

Fachadas  |  Classes  |  Ligação de Container de Serviços
------------- | ------------- | -------------
App  |  [Illuminate\Foundation\Application](http://laravel.com/api/{{version}}/Illuminate/Foundation/Application.html)  | `app`
Artisan  |  [Illuminate\Console\Application](http://laravel.com/api/{{version}}/Illuminate/Console/Application.html)  |  `artisan`
Auth  |  [Illuminate\Auth\AuthManager](http://laravel.com/api/{{version}}/Illuminate/Auth/AuthManager.html)  |  `auth`
Auth (Instance)  |  [Illuminate\Auth\Guard](http://laravel.com/api/{{version}}/Illuminate/Auth/Guard.html)  |
Blade  |  [Illuminate\View\Compilers\BladeCompiler](http://laravel.com/api/{{version}}/Illuminate/View/Compilers/BladeCompiler.html)  |  `blade.compiler`
Bus  |  [Illuminate\Contracts\Bus\Dispatcher](http://laravel.com/api/{{version}}/Illuminate/Contracts/Bus/Dispatcher.html)  |
Cache  |  [Illuminate\Cache\Repository](http://laravel.com/api/{{version}}/Illuminate/Cache/Repository.html)  |  `cache`
Config  |  [Illuminate\Config\Repository](http://laravel.com/api/{{version}}/Illuminate/Config/Repository.html)  |  `config`
Cookie  |  [Illuminate\Cookie\CookieJar](http://laravel.com/api/{{version}}/Illuminate/Cookie/CookieJar.html)  |  `cookie`
Crypt  |  [Illuminate\Encryption\Encrypter](http://laravel.com/api/{{version}}/Illuminate/Encryption/Encrypter.html)  |  `encrypter`
DB  |  [Illuminate\Database\DatabaseManager](http://laravel.com/api/{{version}}/Illuminate/Database/DatabaseManager.html)  |  `db`
DB (Instance)  |  [Illuminate\Database\Connection](http://laravel.com/api/{{version}}/Illuminate/Database/Connection.html)  |
Event  |  [Illuminate\Events\Dispatcher](http://laravel.com/api/{{version}}/Illuminate/Events/Dispatcher.html)  |  `events`
File  |  [Illuminate\Filesystem\Filesystem](http://laravel.com/api/{{version}}/Illuminate/Filesystem/Filesystem.html)  |  `files`
Hash  |  [Illuminate\Contracts\Hashing\Hasher](http://laravel.com/api/{{version}}/Illuminate/Contracts/Hashing/Hasher.html)  |  `hash`
Input  |  [Illuminate\Http\Request](http://laravel.com/api/{{version}}/Illuminate/Http/Request.html)  |  `request`
Lang  |  [Illuminate\Translation\Translator](http://laravel.com/api/{{version}}/Illuminate/Translation/Translator.html)  |  `translator`
Log  |  [Illuminate\Log\Writer](http://laravel.com/api/{{version}}/Illuminate/Log/Writer.html)  |  `log`
Mail  |  [Illuminate\Mail\Mailer](http://laravel.com/api/{{version}}/Illuminate/Mail/Mailer.html)  |  `mailer`
Password  |  [Illuminate\Auth\Passwords\PasswordBroker](http://laravel.com/api/{{version}}/Illuminate/Auth/Passwords/PasswordBroker.html)  |  `auth.password`
Queue  |  [Illuminate\Queue\QueueManager](http://laravel.com/api/{{version}}/Illuminate/Queue/QueueManager.html)  |  `queue`
Queue (Instance) |  [Illuminate\Queue\QueueInterface](http://laravel.com/api/{{version}}/Illuminate/Queue/QueueInterface.html)  |
Queue (Base Class) |  [Illuminate\Queue\Queue](http://laravel.com/api/{{version}}/Illuminate/Queue/Queue.html)  |
Redirect  |  [Illuminate\Routing\Redirector](http://laravel.com/api/{{version}}/Illuminate/Routing/Redirector.html)  |  `redirect`
Redis  |  [Illuminate\Redis\Database](http://laravel.com/api/{{version}}/Illuminate/Redis/Database.html)  |  `redis`
Request  |  [Illuminate\Http\Request](http://laravel.com/api/{{version}}/Illuminate/Http/Request.html)  |  `request`
Response  |  [Illuminate\Contracts\Routing\ResponseFactory](http://laravel.com/api/{{version}}/Illuminate/Contracts/Routing/ResponseFactory.html)  |
Route  |  [Illuminate\Routing\Router](http://laravel.com/api/{{version}}/Illuminate/Routing/Router.html)  |  `router`
Schema  |  [Illuminate\Database\Schema\Blueprint](http://laravel.com/api/{{version}}/Illuminate/Database/Schema/Blueprint.html)  |
Session  |  [Illuminate\Session\SessionManager](http://laravel.com/api/{{version}}/Illuminate/Session/SessionManager.html)  |  `session`
Session (Instance)  |  [Illuminate\Session\Store](http://laravel.com/api/{{version}}/Illuminate/Session/Store.html)  |
Storage  |  [Illuminate\Contracts\Filesystem\Factory](http://laravel.com/api/{{version}}/Illuminate/Contracts/Filesystem/Factory.html)  |  `filesystem`
URL  |  [Illuminate\Routing\UrlGenerator](http://laravel.com/api/{{version}}/Illuminate/Routing/UrlGenerator.html)  |  `url`
Validator  |  [Illuminate\Validation\Factory](http://laravel.com/api/{{version}}/Illuminate/Validation/Factory.html)  |  `validator`
Validator (Instance)  |  [Illuminate\Validation\Validator](http://laravel.com/api/{{version}}/Illuminate/Validation/Validator.html) |
View  |  [Illuminate\View\Factory](http://laravel.com/api/{{version}}/Illuminate/View/Factory.html)  |  `view`
View (Instance)  |  [Illuminate\View\View](http://laravel.com/api/{{version}}/Illuminate/View/View.html)  |
