# Extending The Framework

- [Gerenciadores & Factories(Fábricas)](#managers-and-factories)
- [Cache](#cache)
- [Sessões](#session)
- [Autenticação](#authentication)
- [Conteiner de Serviço Baseado em Extensão](#container-based-extension)

<a name="managers-and-factories"></a>
## Gerenciadores & Factories(Fábricas)

Laravel tem várias classes `gerenciadoras` que gerenciam a criação de drivers baseados-em-componentes. Estes incluem o cache, seção, autenticação, e componentes de queue(fila). A classe gerenciadora é responsável pela criação de uma implementação de driver específico baseado na configuração da aplicação. Por exemplo, a classe `CacheManager` pode criar APC, Mencached, Arquivos, e várias outras implementações de drivers de cache.

Cada um dos gerenciadores incluem um método `extend` que pode ser usado para facilmente a injetar de novas funcionalidades de resolução de drivers no gerenciador. Nos iremos cobrir cada desses gerenciadores abaixo, com exemplos de como injetar um driver de suporte customizado em cada um deles.   

>**Nota:** Pegue um momento para explorar várias classes `gerenciadoras` que vem com o Laravel, como o `CacheManager` e a `SessionManager`. Lendo por essas classes dará um entendimento completo a mais de como Laravel trabalha por baixo dos panos. Todas as classes gerenciadoras extendem da classe base  `Illuminate\Support\Manager, que fornece algumas coisas úteis, funcionalidade comum para cada gerenciador. 

<a name="cache"></a>
## Cache

Para extender da instalações do cache do laravel, nos iremos usar o método `extend` no `CacheManager`, que é usado para vincular um driver resolvedor customizado a um gerenciador, e é comum a todas as classes gerenciadoras. Por exemplo, para o registrar um novo driver cache chamado "mongo", nos podemos fazer o seguinte: 

	Cache::extend('mongo', function($app)
	{
		return Cache::repository(new MongoStore);
	});

O primeiro argumento passado para o método `extend` é o nome do driver. Isto irá corresponder a sua opção `driver` no seu arquivo de configuração `config/cache.php`. O segundo argumento é um Closure que deve retornar uma instância de `Illuminate\Cache\Repository`. A Closure será passada à uma instância de `$app`, que é uma instância de `Illuminate\Foundation\Application` e um container de serviços.

A chamada para `Cache::extend` pode ser realizada no método `boot` do padrão `App\Providers\AppServiceProvider` que vem em instalações nativas de aplicações Larave, ou você pode criar seu próprio fornecedor de serviço para alojar a extenção - apenas não se esqueça de registrar o fornecedor, no array provider no arquivo `config/app.php`.

Para criar nosso driver cache customizado, nos primeiro precisamos implementar o contrato `Illuminate\Contracts\Cache\Store`. Então, nossa implementação de cache MongoDB deve parecer com isto:

	class MongoStore implements Illuminate\Contracts\Cache\Store {

		public function get($key) {}
		public function put($key, $value, $minutes) {}
		public function increment($key, $value = 1) {}
		public function decrement($key, $value = 1) {}
		public function forever($key, $value) {}
		public function forget($key) {}
		public function flush() {}

	}

Nos apenas precisamos implementar cada um destes métodos usando uma conexão MongoDB. Uma vez que nossa implementação é completa, nos podemos finalizar o registro do nosso driver customizado. 

	Cache::extend('mongo', function($app)
	{
		return Cache::repository(new MongoStore);
	});

Se você estiver pensativo sobre aonde alocar o código do seu driver de cache customizado, considere fazer com que o mesmo esteja disponível no Packagist! OU, você pode criar o namespace `Extensions` (Extenções) dentro do seu diretório `app`. No entanto, tenha em mente que o Laravel não tem uma estrutura de aplicação rígida e você é livre para organizar a sua aplicação de acordo com suas preferências. 

<a name="session"></a>
## Sessões 

Extender do Laravel com um driver de sessão customizado é tão fácil quando extender do sistema de cache. Novamente, nos iremos usar o método `extend` para registrar nosso código customizado.

	Session::extend('mongo', function($app)
	{
		// Return implementation of SessionHandlerInterface
	});

### Onde Extender A Sessão

Você deve alocar seu código de extenção da sessão no método `boot` do seu `AppServiceProvider`.

### Escrevendo A extensão Session (Sessão)

Note que o nosso driver se sessão customizado deve implementar a `SessionHandlerInterface`. Esta interface contém apenas alguns simples métodos que nos precisamos implementar. Uma implementação MongoDB encurtada deve parece com algo assim:
	
	class MongoHandler implements SessionHandlerInterface {

		public function open($savePath, $sessionName) {}
		public function close() {}
		public function read($sessionId) {}
		public function write($sessionId, $data) {}
		public function destroy($sessionId) {}
		public function gc($lifetime) {}

	}

Desde que estes métodos não são tão compreensíveis a leitura quanto os da cache `StoreInterface`, vamos rapidamenta explicar o que cada um destes métodos faz:

- O método `open` deve tipicamente ser usado em um arquivo baseado nos sistemas de armazenamento de sessão. Desde que o Laravel entrega o driver da sessão com um `arquivo`, você quase nunca precisará colocar qualquer coisa neste método. Você pode deixar isto em um stub vazio. Isto é pelo simples fato de um design de interface pobre (que nos iremos discutir posteriormente) que o PHP requere que nos façamos a implementação deste método.  
- O método `close`, como o método `open`, pode também ser usualmente desconsiderado. Para a maioria dos drivers, isto não é necessário.
- O método `read` deve retornar a versão string dos dados da sessão associados com o `$sessionId` dado. Não há necessidado para qualquer serealizaççao ou outra codificação quando se esta recuperando ou armazenando dados de sessão no seu driver, já que o Laravel fará a serialização para você.
- O método `write` deve escrever a string `$data` dada associada com o `$sessionId` para algum sistema de armazenamento de persistênte, tais como MongoDB, Dynamo, etc.
- O método `destroy` deve remover os dados associados com o `$sessionId` do armazenamento persistente.
- O método `gc` deve destroir todos os dados da sessão que é mais velho do que `$lifetime`(tempo-de-vida) dado, que é um timestamp UNIX. Para sistemas auto-expiração como Memcached e Redis, este método pode ser deixado vazio. 

Uma vez que a `SessionHandlerInterface` tiver sido implementada, nos estaremos prontos para registrar isto com o gerenciador de Sessão:

	Session::extend('mongo', function($app)
	{
		return new MongoHandler;
	});

Uma vez que o driver de sessão tiver sido registrado, nos podemos usar o driver `mongo` nos seu arquivo de configuração `config/session.php`.

> **Nota:** Lembre-se, se você escrever um manipulador de sessão customizado, compartilhe isso no Packagist!

<a name="authentication"></a>
## Autenticação

Autenticação pode ser extendida do mesma forma que as instalações de cache e sessão. Novamente, no iremos usar o método `extend` que nós nos tornamos familiarizados com: 

	Auth::extend('riak', function($app)
	{
		// Return implementation of Illuminate\Contracts\Auth\UserProvider
	});


As implementações de `UserProvider` são apenas responsáveis por buscar a implementação `Illuminate\Contracts\Auth\Authenticatable` fora de um sistema de armazenamento persistente, como MySQL, Riak, etc. Estas duas interfaces permitem o mecanismo de autenticação do Laravel continuar funcionando independentemente da como os dados do usuário é armazenado ou que tipo de classe é usado para reprensentar isto.

Vamos dar uma olhada no contrato `UserProvider`:

	interface UserProvider {

		public function retrieveById($identifier);
		public function retrieveByToken($identifier, $token);
		public function updateRememberToken(Authenticatable $user, $token);
		public function retrieveByCredentials(array $credentials);
		public function validateCredentials(Authenticatable $user, array $credentials);

	}

A função `retrieveById` típicamente recebe uma chave númerica representando o usuário, tanto como um ID auto-incrementing de um banco de dados MySQL. A implementação `Authenticatable` combinando o ID deve ser recuperado e retornado pelo método. 

A função `retrieveByToken` recupera um usuário pelo seu único `$identifier` e pelo `$token` "Lembre-me", armazenado em um campo `remember_token`. Como o método prévido, a implemtanção de `Authenticatable` deve ser retornado. 

O método `updateRememberToken` atualiza o campo `remember_token` do (usuário) `$user` com um novo `$token`. O novo token pode ser tanto um token recém gerado, atribuído em uma tentativa de login "Lembre-me" de sucesso, ou um usuário nulo é deslogado. 

O método `retrieveByCredentials` recebe um arrya de credenciais passados para o método `Auth::attempt` quando se tenta entrar na aplicação. O método deve então "consultado(query)" o armazenamento persistente subjacente para verificar as credenciais. Tipicamente, este método irá executar uma query com a condição "where"(onde) no `$credentials['username']`. O método deve então retornar a implementação do `UserInterface`. ** Este método não deve tentar fazer qualquer avaliação de senha ou atenticação.**

O método `validateCredentials` deve comparar o dado `$user`(usuário) com as `$credentials`(credenciais) para autenticar o usuário. Por exemplo, este método pode compara a string `$user->getAuthPassword()` ao `Hash::make` do `$credentials['password']`. Este método deve apenas validar as credenciais do usuário e retornar um boolean.

Agora que nos temos explorado cada um dos méotodos no `UserProvider`, vamos dar uma olhada na `Authenticatable`. Lembre-se, o fornecedor deve retornar implementações desta interface a partir dos métodos `retrieveById` e `retrieveByCredentials`:

	interface Authenticatable {

		public function getAuthIdentifier();
		public function getAuthPassword();
		public function getRememberToken();
		public function setRememberToken($value);
		public function getRememberTokenName();

	}

This interface is simple. The `getAuthIdentifier` method should return the "primary key" of the user. In a MySQL back-end, again, this would be the auto-incrementing primary key. The `getAuthPassword` should return the user's hashed password. This interface allows the authentication system to work with any User class, regardless of what ORM or storage abstraction layer you are using. By default, Laravel includes a `User` class in the `app` directory which implements this interface, so you may consult this class for an implementation example.

Finally, once we have implemented the `UserProvider`, we are ready to register our extension with the `Auth` facade:

	Auth::extend('riak', function($app)
	{
		return new RiakUserProvider($app['riak.connection']);
	});

After you have registered the driver with the `extend` method, you switch to the new driver in your `config/auth.php` configuration file.

<a name="container-based-extension"></a>
## Conteiner de Serviço Baseado em Extensão

Quase todos os fornecedores de serviços incluídos com o famework Laravel combinam objetos em um container de serviços. Você pode achar a lista dos fornecedores de serviço da sua aplicação no seu arquivo de configuração `config/app.php`. Como você tem tempo, você deve percorrer pelo código fonte de cada um desses fornecedores de serviço. Ao fazer isto, você irá ganhar um melhor entendimento do que cada fornecedor adiciona ao framework, bem como quais chaves são usadas para combinar vários serviços em um container de serviço.


Por exemplo, o `HashServiceProvider` combina uma chave `hash` em um container de serviço, que resolve na instância de `Illuminate\Hashing\BcryptHasher`. Você pode facilmente extender e sobrescrever esta classe dentro da sua aplicação sobrescrevendo esta combinação. Por exemplo:

	<?php namespace App\Providers;

	class SnappyHashProvider extends \Illuminate\Hashing\HashServiceProvider {

		public function boot()
		{
			parent::boot();
					
			$this->app->bindShared('hash', function()
			{
				return new \Snappy\Hashing\ScryptHasher;
			});
		}

	}

Note que esta classe extende a `HashServiceProvider`, e não a classe base padrão`ServiceProvider`. Uma vez que você tenha extendido o fornecedo de serviço, troque o
`HashServiceProvider` no seu arquivo de configuração `config/app.php` com o nome do seu fornecedor de serviço extendido.

Isto é o método geral de extensão de qualquer classe do core(núcleo) que está ligada ao container. Essencialmente toda classe de core(núcleo) é ligada ao container desta forma, e pode ser sobrescrita. Novamente, a leitura através dos fornecedores de serviços incluídos no framework irão famialirizar você com onde várias classe são ligadas ao container, e quais chaves por quais eles estão ligados. Este é um grande método para aprender mais sobre como os componentes do Laravel são unidos.
