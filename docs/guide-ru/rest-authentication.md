Аутентификация
==============

В отличие от веб-приложений, RESTful API обычно не сохраняют информацию о состоянии, а это означает, что сессии и куки
использовать не следует. Следовательно, раз состояние аутентификации пользователя не может быть сохранено в сессиях или куках,
каждый запрос должен приходить вместе с определенным видом параметров аутентификации. Общепринятая практика состоит в том,
что для аутентификации пользователя с каждым запросом отправляется секретный токен доступа. Так как токен доступа
может использоваться для уникальной идентификации и аутентификации пользователя, **запросы к API всегда должны отсылаться
через протокол HTTPS, чтобы предотвратить атаки «человек посередине» (англ. "man-in-the-middle", MitM)**.

Есть различные способы отправки токена доступа:

* [HTTP Basic Auth](http://en.wikipedia.org/wiki/Basic_access_authentication): токен доступа
  отправляется как имя пользователя. Такой подход следует использовать только в том случае, когда токен доступа может быть безопасно сохранен
  на стороне пользователя API. Например, если API используется программой, запущенной на сервере.
* Параметр запроса: токен доступа отправляется как параметр запроса в URL-адресе API, т.е. примерно таким образом:
  `https://example.com/users?access-token=xxxxxxxx`. Так как большинство веб-серверов сохраняют параметры запроса в своих логах,
  такой подход следует применять только при работе с `JSONP`-запросами, которые не могут отправлять токены доступа
 в HTTP-заголовках.
* [OAuth 2](http://oauth.net/2/): токен доступа выдается пользователю API сервером авторизации
  и отправляется API-серверу через [HTTP Bearer Tokens](https://datatracker.ietf.org/doc/html/rfc6750),
  в соответствии с протоколом OAuth2.

Yii поддерживает все выше перечисленные методы аутентификации. Вы также можете легко создавать новые методы аутентификации.

Чтобы включить аутентификацию для ваших API, выполните следующие шаги:

1. У [компонента приложения](structure-application-components.md) `user` установите свойство
   [[yii\web\User::enableSession|enableSession]] равным `false`.
2. Укажите, какие методы аутентификации вы планируете использовать, настроив поведение `authenticator`
   в ваших классах REST-контроллеров.
3. Реализуйте метод [[yii\web\IdentityInterface::findIdentityByAccessToken()]] в вашем [[yii\web\User::identityClass|классе UserIdentity]].

Шаг 1 не обязателен, но рекомендуется его всё-таки выполнить, так как RESTful API не должен сохранять информацию о
состоянии клиента. Когда свойство [[yii\web\User::enableSession|enableSession]] установлено в `false`, состояние
аутентификации пользователя НЕ БУДЕТ сохраняться между запросами с использованием сессий. Вместо этого аутентификация
будет выполняться для каждого запроса, что достигается шагами 2 и 3.

> Tip: если вы разрабатываете RESTful API в пределах приложения, вы можете настроить свойство
> [[yii\web\User::enableSession|enableSession]] компонента приложения `user` в конфигурации приложения. Если вы
> разрабатываете RESTful API как модуль, можете добавить следующую строчку в метод `init()` модуля:
>
> ```php
> public function init()
> {
>     parent::init();
>     \Yii::$app->user->enableSession = false;
> }
> ```

Например, для использования HTTP Basic Auth, вы можете настроить свойство `authenticator` следующим образом:

```php
use yii\filters\auth\HttpBasicAuth;

public function behaviors()
{
    $behaviors = parent::behaviors();
    $behaviors['authenticator'] = [
        'class' => HttpBasicAuth::class,
    ];
    return $behaviors;
}
```

Если вы хотите включить поддержку всех трёх описанных выше методов аутентификации, можете использовать `CompositeAuth`:

```php
use yii\filters\auth\CompositeAuth;
use yii\filters\auth\HttpBasicAuth;
use yii\filters\auth\HttpBearerAuth;
use yii\filters\auth\QueryParamAuth;

public function behaviors()
{
    $behaviors = parent::behaviors();
    $behaviors['authenticator'] = [
        'class' => CompositeAuth::class,
        'authMethods' => [
            HttpBasicAuth::class,
            HttpBearerAuth::class,
            QueryParamAuth::class,
        ],
    ];
    return $behaviors;
}
```

Каждый элемент в массиве `authMethods` должен быть названием класса метода аутентификации или массивом настроек.


Реализация метода `findIdentityByAccessToken()` определяется особенностями приложения. Например, в простом варианте,
когда у каждого пользователя есть только один токен доступа, вы можете хранить этот токен в поле `access_token`
таблицы пользователей. В этом случае метод `findIdentityByAccessToken()` может быть легко реализован в классе `User` следующим образом:

```php
use yii\db\ActiveRecord;
use yii\web\IdentityInterface;

class User extends ActiveRecord implements IdentityInterface
{
    public static function findIdentityByAccessToken($token, $type = null)
    {
        return static::findOne(['access_token' => $token]);
    }
}
```

После включения аутентификации описанным выше способом при каждом запросе к API запрашиваемый контроллер
будет пытаться аутентифицировать пользователя в своем методе `beforeAction()`.

Если аутентификация прошла успешно, контроллер выполнит другие проверки (ограничение частоты запросов, авторизация)
и затем выполнит действие. Информация об аутентифицированном пользователе может быть получена из объекта `Yii::$app->user->identity`.

Если аутентификация прошла неудачно, будет возвращен ответ с HTTP-кодом состояния 401 вместе с другими необходимыми заголовками
(такими, как заголовок `WWW-Authenticate` для HTTP Basic Auth).


## Авторизация <span id="authorization"></span>

После аутентификации пользователя вы, вероятно, захотите проверить, есть ли у него или у неё разрешение на выполнение запрошенного
действия с запрошенным ресурсом. Этот процесс называется *авторизацией* и подробно описан
в разделе «[Авторизация](security-authorization.md)».

Если ваши контроллеры унаследованы от [[yii\rest\ActiveController]], вы можете переопределить
метод [[yii\rest\Controller::checkAccess()|checkAccess()]] для выполнения авторизации. Этот метод будет вызываться
встроенными действиями, предоставляемыми контроллером [[yii\rest\ActiveController]].
