git bb3c5b869472e1d3f6df6538a7beb9f3be8e2826

---

# Интерфейс командной строки Artisan

- [Введение](#introduction)
- [Создание команд](#writing-commands)
    - [Структура команды](#command-structure)
- [Ввод и вывод данных](#command-io)
    - [Определение ожидаемых данных](#defining-input-expectations)
    - [Получение данных ввода](#retrieving-input)
    - [Запрос на ввод данных](#prompting-for-input)
    - [Вывод данных](#writing-output)
- [Регистрация команд](#registering-commands)
- [Запуск команд из кода](#calling-commands-via-code)

<a name="introduction"></a>
## Введение

Artisan — название интерфейса командной строки, входящего в состав Laravel. Он предоставляет полезные команды для использования во время разработки вашего приложения. Работает на основе мощного компонента — Symfony Console. Для вывода списка всех доступных команд Artisan вы можете использовать команду `list`:

    php artisan list

Каждая команда также включает и инструкцию, которая отображает и описывает доступные аргументы и опции для команды. Для того, чтобы её вывести, просто добавьте слово `help` перед командой:

    php artisan help migrate

<a name="writing-commands"></a>
## Создание команд

Вы также можете создавать собственные команды Artisan, специфичные для вашего приложения. Их можно хранить в папке `app/Console/Commands`; тем не менее, вы вольны выбрать любое другое местоположение, основанное на настройках автозагрузки в `composer.json`.

Чтобы создать новую команду, нужно воспользоваться командой `make:console`, которая сгенерирует шаблон команды для старта:

    php artisan make:console SendEmails

Команда выше сгенерирует класс `app/Console/Commands/SendEmails.php`. При создании команды вы можете указать опцию `--command`, которая будет использоваться для присвоения команде имени:

    php artisan make:console SendEmails --command=emails:send

<a name="command-structure"></a>
### Структура команды

После создания команды, вы должны заполнить свойства `signature` и `description` созданного класса, которые будут использованы для отображения вашей команды в списке(`list`) команд.

Метод `handle` будет вызываться при запуске вашей команды. Вы можете размещать там любую требуемую логику. Давайте посмотрим на примере.

Заметьте, что мы можем внедрить любую зависимость в конструктор команды. [Сервис контейнер](/docs/{{version}}/container) Laravel автоматически внедрит все указанные зависимости в конструктор. Для большего реиспользования кода хорошей практикой будет написание ваших команд «лёгкими» и выполнение всех необходимых действий в сервисах вашего приложения.

    <?php

    namespace App\Console\Commands;

    use App\User;
    use App\DripEmailer;
    use Illuminate\Console\Command;
    use Illuminate\Foundation\Inspiring;

    class Inspire extends Command
    {
        /**
         * The name and signature of the console command.
         *
         * @var string
         */
        protected $signature = 'email:send {user}';

        /**
         * The console command description.
         *
         * @var string
         */
        protected $description = 'Send drip e-mails to a user';

        /**
         * The drip e-mail service.
         *
         * @var DripEmailer
         */
        protected $drip;

        /**
         * Create a new command instance.
         *
         * @param  DripEmailer  $drip
         * @return void
         */
        public function __construct(DripEmailer $drip)
        {
            parent::__construct();

            $this->drip = $drip;
        }

        /**
         * Execute the console command.
         *
         * @return mixed
         */
        public function handle()
        {
            $this->drip->send(User::find($this->argument('user')));
        }
    }

<a name="command-io"></a>
## Ввод и вывод данных

<a name="defining-input-expectations"></a>
### Определение ожидаемых данных

При написании консольных команд стандартной практикой является получение данных от пользователя при помощи аргументов и опций. Laravel предоставляет удобный способ определения аргументов, ожидаемых от пользователя, путём использования свойства `signature` команды. Это свойство позволяет вам определить название, аргументы и опции команды при помощи простого и понятного синтаксиса.

Ожидаемые аргументы и опции от пользователя обрамляются в фигурные скобки, например:

    /**
     * The name and signature of the console command.
     *
     * @var string
     */
    protected $signature = 'email:send {user}';

Вы также можете делать аргументы необязательными или определять значения по умолчанию:

    // Необязательный аргумент
    email:send {user?}

    // Необязательный аргумент со значением по умолчанию
    email:send {user=foo}

Опции, как и аргументы, — ещё один способ получения данных от пользователя. Они отделяются двумя дефисами (`--`) при указании их в строке названия команды. Мы можем определить опции так:

    /**
     * The name and signature of the console command.
     *
     * @var string
     */
    protected $signature = 'email:send {user} {--queue}';

В этом примере опция `--queue` может быть указана при вызове команды. Если опция будет указана, то её значение будет `true`. Иначе — `false`:

    php artisan email:send 1 --queue

Вы также можете указать для опции вводимое значение при помощи знака равно `=` после опции:

    /**
     * The name and signature of the console command.
     *
     * @var string
     */
    protected $signature = 'email:send {user} {--queue=}';

В этом примере пользователь может указать значение опции так:

    php artisan email:send 1 --queue=default

Вы также можете указать значение по умолчанию для опции:

    email:send {user} {--queue=default}

Для опции можно определить шорткат (короткую запись). Для этого укажите все возможные написания опции через символ `|`:

    email:send {user} {--Q|queue}

Если в качестве значений для аргументов или опций у вас будут фигурировать массивы, используйте для их определения символ `*`:

    email:send {user*}

    email:send {user} {--id=*}

#### Описание вводимых данных

Описание для аргументов и опций можно указать путём разделения их и описания знаком двоеточия:

    /**
     * The name and signature of the console command.
     *
     * @var string
     */
    protected $signature = 'email:send
                            {user : The ID of the user}
                            {--queue= : Whether the job should be queued}';

<a name="retrieving-input"></a>
### Получение данных ввода

Очевидно, что во время выполнения команды у вас должен быть доступ к введённым пользователем значениям. Для их получения вы можете воспользоваться методами `argument` и `option`:

Для получения значения аргумента используйте метод `argument`:

    /**
     * Execute the console command.
     *
     * @return mixed
     */
    public function handle()
    {
        $userId = $this->argument('user');

        //
    }

Если вам необходимо получить все аргументы массивом, используйте `argument` без параметров:

    $arguments = $this->argument();

При помощи метода `option` опции можно получить так же просто, как и аргументы. Как и метод `argument`, вы можете вызвать `option` без параметров для получения массива опций:

    // Получение конкретной опции
    $queueName = $this->option('queue');

    // Получение всех опций
    $options = $this->option();

Если аргумент или опция не существуют, будет возвращено `null`.

<a name="prompting-for-input"></a>
### Запрос на ввод данных

В дополнение к отображению вывода, вы также можете запрашивать у пользователя дополнительные данные во время выполнения команды. Метод `ask` будет спрашивать у пользователя заданный вопрос и получать данные, затем возвращая эти данные вашей команде:

    /**
     * Execute the console command.
     *
     * @return mixed
     */
    public function handle()
    {
        $name = $this->ask('What is your name?');
    }

Метод `secret` подобен методу `ask`, но данные, вводимые пользователем, не будут видны в консоли по мере ввода. Этот метод полезен для запроса секретной информации, например, пароля:

    $password = $this->secret('What is the password?');

#### Запрос подтверждения

Если вам нужно спросить пользователя о подтверждении, вы можете воспользоваться методом `confirm`. По умолчанию он будет возвращать `false`. Но если пользователь введёт `y`, метод вернёт `true`.

    if ($this->confirm('Do you wish to continue? [y|N]')) {
        //
    }

#### Предоставление пользователю выбора

Метод `anticipate` может быть использован для отображения возможных вариантов выбора. Пользователь сможет выбрать любой вариант из предоставляемых или ввести свой.

    $name = $this->anticipate('What is your name?', ['Taylor', 'Dayle']);

Если вы хотите дать возможность пользователю выбирать только из предоставляемых вариантов, воспользуйтесь методом `choice`. Пользователь должен будет выбрать номер варианта. Вы можете указать значение по умолчанию, если пользователь ничего не выбрал:

    $name = $this->choice('What is your name?', ['Taylor', 'Dayle'], $default);

<a name="writing-output"></a>
### Вывод данных

Для вывода данных в консоль используйте методы `line`, `info`, `comment`, `question` и `error`. Для каждого метода будет использоваться соответствующий цвет ANSI при выводе.

Для вывода информационного сообщения пользователю используйте метод `info`. Обычно выводимый текст будет отображаться зелёным цветом:

    /**
     * Execute the console command.
     *
     * @return mixed
     */
    public function handle()
    {
        $this->info('Display this on the screen');
    }

Для отображения сообщения об ошибке используйте метод `error`. Эти сообщения обычно отображаются красным цветом:

    $this->error('Something went wrong!');

Для вывода обычного текста используйте метод `line`. Сообщение будет выведено в дефолтном цвете терминала:

    $this->line('Display this on the screen');    

#### Вывод таблиц

Метод `table` делает простым отображения таблиц данных. Просто задайте заголовки столбцов и данные в методе. Ширина и высота будут автоматически вычислены в зависимости от данных:

    $headers = ['Name', 'Email'];

    $users = App\User::all(['name', 'email'])->toArray();

    $this->table($headers, $users);

#### Индикатор прогресса

Для долгих задач может понадобится вывод индикатора прогресса. Используя объект `output`, мы можем стартовать, изменять и останавливать индикатор. Вам необходимо будет определить число шагов в начале задачи и изменять это количество в процессе:

    $users = App\User::all();

    $bar = $this->output->createProgressBar(count($users));

    foreach ($users as $user) {
        $this->performTask($user);

        $bar->advance();
    }

    $bar->finish();

За более продвинутыми опциями вы можете обратиться к документации [Symfony Progress Bar component documentation](http://symfony.com/doc/2.7/components/console/helpers/progressbar.html).

<a name="registering-commands"></a>
## Регистрация команд

После написания команды вам необходимо её зарегистрировать в Artisan, чтобы она была доступна для использования. Это можно сделать в файле `app/Console/Kernel.php`.

В этом файле вы найдёте список команд в свойстве `command`. Для регистрации команды просто добавьте название класса в список. Когда Artisan загружается, все команды проходят через [сервис контейнер](/docs/{{version}}/container) и регистрируются в Artisan:

    protected $commands = [
        Commands\SendEmails::class
    ];

<a name="calling-commands-via-code"></a>
## Запуск команд из кода

Иногда вам может понадобиться запускать команды извне консоли. Например, вы захотите запустить команду в роутере или контроллере. Для этого можно использовать метод `call` фасада `Artisan`. Он принимает название команды первым аргументом и массив параметров команды вторым. После выполнения будет возвращён код выхода:

    Route::get('/foo', function () {
        $exitCode = Artisan::call('email:send', [
            'user' => 1, '--queue' => 'default'
        ]);

        //
    });

Метод `queue` фасада `Artisan` позволяет заносить команды в очередь для выполнения в фоне при помощи [менеджера очередей](/docs/{{version}}/queues):

    Route::get('/foo', function () {
        Artisan::queue('email:send', [
            'user' => 1, '--queue' => 'default'
        ]);

        //
    });

Если опция в вашей команде не предусматривает значений по умолчанию, вы можете передать в неё любое логическое значение, например `true`, чтобы просто обозначить её использование:

    $exitCode = Artisan::call('migrate:refresh', [
        '--force' => true,
    ]);

### Вызов команд из других команд

Иногда вам может понадобиться вызывать другие команды из существующих команд. Вы можете сделать это при помощи всё того же метода `call`:

    /**
     * Execute the console command.
     *
     * @return mixed
     */
    public function handle()
    {
        $this->call('email:send', [
            'user' => 1, '--queue' => 'default'
        ]);

        //
    }

Если вы хотите, чтобы во время выполнения другой команды был скрыт весь её вывод, можно использовать метод `callSilent`. Метод `callSilent` вызывается с теми же параметрами, что и `call`:

    $this->callSilent('email:send', [
        'user' => 1, '--queue' => 'default'
    ]);
