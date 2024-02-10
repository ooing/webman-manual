# О утечках памяти
Webman - это постоянный фреймворк в памяти, поэтому нам нужно немного обратить внимание на утечки памяти. Однако разработчикам не стоит слишком беспокоиться, потому что утечка памяти происходит в крайне редких случаях и легко избежать. Опыт разработки веб-ман и традиционных фреймворков практически идентичен, и нет необходимости делать излишние действия по управлению памятью.

> **Совет**
> Процесс мониторинга, встроенный в веб-ман, будет отслеживать использование памяти всеми процессами. Если использование памяти процессом скоро достигнет установленного в php.ini значения `memory_limit`, соответствующий процесс будет автоматически безопасно перезапущен для освобождения памяти, что не повлияет на бизнес в процессе.

## Определение утечки памяти
С увеличением количества запросов память, используемая Webman, также **бесконечно растет** (обратите внимание - **бесконечно растет**), достигая нескольких сотен мегабайт и даже больше, что является утечкой памяти. Если память увеличивается, но затем перестает увеличиваться, это не считается утечкой памяти.

Обычное использование памяти в несколько десятков мегабайт - это вполне нормальная ситуация. Однако при обработке очень больших запросов или поддержке огромного количество соединений, использование памяти одиночного процесса может достигать нескольких сотен мегабайт, что тоже вполне естественно. После использования этой части памяти PHP возможно не освободит ее полностью для операционной системы. Вместо этого он оставит ее для повторного использования, поэтому может возникнуть ситуация, когда после обработки какого-то большого запроса использование памяти увеличится и не освободится - это нормальное явление. (Вызов метода gc_mem_caches() может освободить часть неиспользуемой памяти.)

## Как происходит утечка памяти
**Утечка памяти происходит, если выполняются следующие два условия:**
1. Существует **длительный жизненный цикл** массива (обратите внимание, что речь идет о **длительном жизненном цикле** массива, а не о обычном массиве)
2. И этот **длительный жизненный цикл** массива будет бесконечно увеличиваться (бизнес бесконечно добавляет в него данные, но не очищает)

Если **одновременно** выполняются условия 1 и 2, то возникнет утечка памяти. В противном случае, если хотя бы одно из указанных условий не выполняется, или выполняется только одно из них, то утечка памяти не произойдет.

## Массив с длительным жизненным циклом

Массивы с длительным жизненным циклом в Webman включают в себя:
1. Массивы со словом static
2. Массивы свойств синглтонов
3. Массивы с глобальным словом

> **Помните**
> В веб-мане разрешается использовать данные с длительным жизненным циклом, но необходимо убедиться, что данные внутри массива ограничены, количество элементов не будет бесконечно расти.

Ниже приведены примеры

#### Бесконечно увеличивающийся статический массив
```php
class Foo
{
    public static $data = [];
    public function index(Request $request)
    {
        self::$data[] = time();
        return response('hello');
    }
}
```

Массив `$data`, определенный со словом `static`, имеет длительный жизненный цикл, и в приведенном примере массив `$data` бесконечно увеличивается с запросами, что приводит к утечке памяти.

#### Бесконечно увеличивающийся массив свойства одиночки
```php
class Cache
{
    protected static $instance;
    public $data = [];
    
    public function instance()
    {
        if (!self::$instance) {
            self::$instance = new self;
        }
        return self::$instance;
    }
    
    public function set($key, $value)
    {
        $this->data[$key] = $value;
    }
}
```

Код вызова
```php
class Foo
{
    public function index(Request $request)
    {
        Cache::instance()->set(time(), time());
        return response('hello');
    }
}
```

`Cache::instance()` возвращает одиночку Cache, которая имеет длительный жизненный цикл классового экземпляра. Хотя его свойство `$data` не объявлено со словом `static`, но так как сам класс имеет длительный жизненный цикл, то `$data` также является массивом с длительным жизненным циклом. При добавлении разных ключей данных в массив `$data` при каждом запросе, использование памяти программы будет только увеличиваться, что приводит к утечке памяти.

> **Помните**
> Если ключ, передаваемый методу Cache::instance()->set(key, value), имеет ограниченное количество, то утечки памяти не будет, потому что массив `$data` не бесконечно увеличивается.

#### Бесконечно увеличивающийся глобальный массив
```php
class Index
{
    public function index(Request $request)
    {
        global $data;
        $data[] = time();
        return response($foo->sayHello());
    }
}
```
Массив, определенный с помощью слова global, не будет освобожден после выполнения функции или метода, поэтому он имеет длительный жизненный цикл. В приведенном выше коде с увеличением запросов произойдет утечка памяти. Также внутри функции или метода массив, определенный со словом static, также является массивом с длительным жизненным циклом, и если массив бесконечно увеличивается, это также вызовет утечку памяти, например:

```php
class Index
{
    public function index(Request $request)
    {
        static $data = [];
        $data[] = time();
        return response($foo->sayHello());
    }
}
```

## Рекомендации
Разработчикам рекомендуется не обращать особого внимания на утечки памяти, потому что это происходит крайне редко, и если несчастье все же произойдет, мы сможем определить утечку с помощью тестирования производительности и тем самым найти проблемное место. Даже если разработчики не найдут утечку памяти, сервис мониторинга webman встроенный в него, своевременно перезапустит процесс, в котором произошла утечка памяти, для освобождения памяти.

Если все-таки вы хотите избежать утечек памяти, вы можете обратить внимание на следующие рекомендации.
1. Стремитесь не использовать массивы со словами `global` и `static`, и если используете, убедитесь, что они не будут бесконечно увеличиваться.
2. Для незнакомых классов по возможности избегайте применения синглтонов, используйте ключевое слово `new` для их инициализации. Если требуется синглтон, проверьте, имеет ли он массив свойств, который будет бесконечно увеличиваться.