# WebApiTemplate
Шаблон-заготовка для приложения WEB API на AspNetCore6. Настроено журналирование с репликацией файлов лога, привязка конфигурации проекта к файлу appsettings.json, CORS политики, размещение в качестве службы Windows.

Обычно моя работа связана с разработкой функционала REST веб-сервисов. Чаще всего, разработка эта ведется на базе уже существующих приложений, созданных и настроенных ранее по шаблону Web API в VisualStudio. Создавать новые приложения приходится не часто, последнее созданное мной, было еще на .NET Core 3.1, поэтому, когда возникает подобная задача, приходится тратить время на повторное изучение технологий первоначальной настройки приложения, чтобы оно отвечало всем требованиям бизнес-процесса компании, в которой я работаю. Столкнулся с этой задачей накануне, решил создать шаблон приложения (ссылка на репозиторий GitHub), в котором уже все настроено и готово. Краткое описание процесса привожу в этой статье. Постарался разбить сам процесс на независимые блоки, чтобы для реализации одного из них не приходилось изучать другие. Намеренно подробно освящаю настройку базовых функции, не вдаваясь в описание принципа работы той или иной функции - для более глубокого понимания привожу ссылки на документацию, по которой учился сам. Статья моя будет полезна для новичков в качестве отправной точки для изучения тех или иных функций .Net Core 6, а так же для специалистов, которые как и я, сосредоточены на реализации бизнес-логики приложения и требуется сократить время восстановления в памяти технологии его первоначальной настройки.

Функционал, который требуется настроить
Журналирование в файл с репликацией лога

Хранение и передачу настроек приложения посредством механизма внедрения зависимостей

Настройка CORS политик для доступа к API посредством кросс-доменных запросов

Настройка Kestrel на прослушивание определенного порта TCP в режиме публикации селф-хостинг

Публикация приложения в качестве службы Windows

Создание сборки
Сборка, в соответствии с документацией, должна включать в себя два приложения:

Приложение внешнего слоя - собственно сам Web API с контроллерами и всеми настройками. Это приложение в VisualStudio создается по шаблону "Asp .Net Core Web API" с установкой опций "Use controllers" и "Enable Open API support". Последняя добавляет в проект автодокументацию на swagger - невероятно полезная штука для отладки

Приложение уровня ядра для реализации бизнес-логики. Представляет собой библиотеку классов, созданную по шаблону "Class Library"

Настройка работы Kestrel для прослушивания определенного порта в приложении aspnetcore 6
В файле конфигурации appsettings.json
Используется для указания TCP порта встроенному веб-серверу Kestrel при публикации приложения. Добавить параметр Urls:

...
  "Urls": "http://*:5005"
...
В файле конфигурации запуска приложения Properties\launchSettings.json
Используется для запуска Swagger в режиме отладки. Изменить порт в параметре applicationUrl профиля для запуска приложения в режиме селф-хост:

...
"profiles": {
    "WebApiTemplate": {
      "commandName": "Project",
      "dotnetRunMessages": true,
      "launchBrowser": true,
      "launchUrl": "swagger",
      "applicationUrl": "http://localhost:5005",
      "environmentVariables": {
        "ASPNETCORE_ENVIRONMENT": "Development"
      }
    },
...
После этих настроек, приложение можно запустить в режиме отладки и убедиться, что swagger доступен на порту 5005 и запрос на созданный по умолчанию метод Get в контроллере WeatherForecast доступен и выполняется на 5005 порту:


Внедрение зависимостей в приложении aspnetcore 6
Документация:

Статья на сайте Metanit, спасибо ребятам за толковую документацию

В Program.cs вносим в коллекцию служб приложения требуемый тип. Добавляем строку в любом месте файла между объявлением builder и вызовом его метода сборки Web приложения var app = builder.Build();:

builder.Services.AddTransient<WebApiService>();
Различные варианты жизненных циклов внедряемых зависимостей отлично описаны в документации, приведу только одну, не очевидную, на первый взгляд, ситуацию. Если требуется создать singleton, то такое объявление:

builder.Services.AddSingleton<ISomeClass, SomeClass>();
не приведет к незамедлительному созданию экземпляра класса, он будет создан только тогда, когда произойдет первое создание экземпляра другого класса, который ссылается зависимостью на этот. Для того, чтобы синглтон создавался незамедлительно при старте приложения, объявлять его нужно так:

builder.Services.AddSingleton<ISomeClass>(new SomeClass());
Но тут могут возникнуть трудности, если конструктор класса имеет параметры и тоже требует внедрения зависимостей. Все эти зависимости можно при желании получить из builder.

Передача настроек appsettings.json через внедрение зависимостей в приложении aspnetcore 6
Документация:

Статья на сайте Metanit, спасибо ребятам за толковую документацию

Создаем класс для настроек (ApplicationSettings)

Подключаем настройки в Program.cs. Добавляем строку в любом месте файла между объявлением builder и вызовом его метода сборки Web приложения var app = builder.Build();:

builder.Services.Configure<ApplicationSettings>(builder.Configuration);
В классе, где требуется экземпляр объекта настроек, получаем значение через внедрение зависимостей:

при помощи NuGet, ставим пакет Microsoft.Extensions.Options

объявляем readonly поле:

private readonly ApplicationSettings _settings;
внедряем зависимость через конструктор:

public WebApiService(IOptions<ApplicationSettings> options)
{
    _settings = options.Value ?? throw new ArgumentNullException("ApplicationSettings");
}
Настройка NLog в приложении aspnetcore 6
В нашей компании так заведено, что все функции журналирования реализованы с использованием пакета NLog. Эта традиция живет у нас со времен .Net Framework, возможно, в данный момент, в функции журналирования .NET Core уже встроены все преимущества NLog, но мне это не известно, поэтому в данной статье рассматривается именно такой способ.

Документация:

Быстрый запуск

Конфигурация

Подключаем NLog в Program.cs. Добавляем строки в любом месте файла между объявлением builder и вызовом его метода сборки Web приложения var app = builder.Build();:

	// NLog: Setup NLog for Dependency injection
    builder.Logging.ClearProviders();
    builder.Host.UseNLog();
В классе, где требуется журналирование, устанавливаем нужные пакеты, внедряем логгер, зависимость, назначив ему соответствующую категорию:

при помощи NuGet, если требуется, ставим пакет Microsoft.Extensions.Logging.Abstractions

объявляем readonly поле:

private readonly ILogger<WebApiService> _logger; //здесь "WebApiService" - категория журнала, тип, от имени которого будут поступать сообщения
внедряем зависимость через конструктор:

public WebApiService(ILogger<WebApiService> logger)
{
    _logger = logger;
}
Запись в журнал:

_logger.LogDebug($"{logText}, значение параметра из настроек: {_settings.SomeParameter}");
Настраиваем цели для журналирования, категории, репликацию лог файлов, в конфигурации nlog.config. Размещенный в репозитории файл конфигурации уже содержит настройку репликации, при которой хранятся лог-файлы за последние 10 дней. Запись лога в файл настроена так, чтобы не забивало журнал сообщениями используемых библиотек Microsoft.

Проверка. Запускаем приложение, вызываем метод http://localhost:5005/WeatherForecast. Результат работы приложения из приведенного выше репозитория из файла журнала:

2023-01-06 13:10:45.2375 | INFO | Microsoft.AspNetCore.Hosting.Diagnostics | Request starting HTTP/1.1 GET http://localhost:5005/WeatherForecast - - | 
2023-01-06 13:10:45.3266 | DEBUG | WebApiTemplate.Controllers.WeatherForecastController | Проверяем внедрение зависимостей, значение параметра из настроек: Some value | 
2023-01-06 13:10:45.3266 | DEBUG | WebApiTemplate.Core.Services.WebApiService | Проверяем работу сервиса уровня ядра, значение параметра из настроек: Some value | 
2023-01-06 13:10:45.3266 | INFO | WebApiTemplate.Controllers.WeatherForecastController | Action status: Ok | 
2023-01-06 13:10:45.3945 | INFO | Microsoft.AspNetCore.Hosting.Diagnostics | Request finished HTTP/1.1 GET http://localhost:5005/WeatherForecast - - - 200 - application/json;+charset=utf-8 157.2347ms |
В журнале видно поступление запроса на контроллер, начало выполнения метода WebApiTemplate.Controllers.WeatherForecastController.Get, вызов метода сервиса WebApiTemplate.Core.Services.WebApiService.SomeAction, результат работы этого метода и отчет о завершении запроса.

Настройка CORS политик в приложении aspnetcore 6
Документация:

Статья на сайте Metanit по конфигурации CORS, спасибо ребятам за толковую документацию

Статья на сайте Metanit по настройке политик CORS, спасибо ребятам за толковую документацию

Для моих производственных нужд обычно требуется два набора политик: для отладки (разрешено все) и для публикации (разрешено только то, что требуется)

В appsettings.json создаем свойство AllowedOrigins:

{
  ...
  "AllowedOrigins": [
    "http://localhost:8080"
  ]
}
добавляем параметр в объект настроек

/// <summary>
/// Настройки приложения
/// </summary>
public class ApplicationSettings
{
    ...
    /// <summary>
    /// Разрешенные домены - источники запросов для политики CORS
    /// </summary>
    public string[] AllowedOrigins { get; set; }
}
Так как потребуется получение настроек, в Program.cs создаем экземпляр объекта настроек в любом месте файла между объявлением builder и вызовом его метода сборки Web приложения var app = builder.Build();:

var settings = builder.Configuration.Get<ApplicationSettings>() ?? throw new ArgumentNullException("ApplicationSettings", "Конфигурация не получена");
В Program.cs объявляем CORS политики до вызова builder.Build()

    builder.Services.AddCors(options => options.AddPolicy("AllowAny", builder => builder
        .AllowAnyOrigin()
        .AllowAnyHeader()
        .AllowAnyMethod())
    );

    builder.Services.AddCors(options => options.AddPolicy("AllowSome", builder => builder
        .WithOrigins(settings.AllowedOrigins)
        .AllowAnyHeader()
        .AllowAnyMethod()
        .AllowCredentials())
    );
В зависимости от типа запуска приложения (Debug или Release), подключаем ту или иную политику в файле Program.cs между var app = builder.Build(); и app.Run();:

#if DEBUG
    app.UseCors("AllowAny");
#else
    app.UseCors("AllowSpecURLs");
#endif
Настройка публикации приложения aspnetcore 6 в качестве службы Windows
Если попытаться запустить приложение, как службу, менеджер служб WIndows, ожидая статус запуска из параметров stdout, не получив этот статус, выдаст ошибку "Ошибка 1053 Служба не ответила на запрос своевременно". Для устранения этой ошибки требуется дополнительная настройка приложения.

Документация:

Статья MSDN на русском

Устанавливаем пакет Microsoft.Extensions.Hosting.WindowsServices

в Program.cs создаем объект настроек и передаем его в конструктор builder взамен строки var builder = WebApplication.CreateBuilder(args);

var options = new WebApplicationOptions
{
    Args = args,
    ContentRootPath = WindowsServiceHelpers.IsWindowsService()
        ? AppContext.BaseDirectory : default
};
var builder = WebApplication.CreateBuilder(options);
Публикуем приложение и размещаем его на сервере Windows любым способом

На сервере любым доступным способом регистрируем и запускаем службу
