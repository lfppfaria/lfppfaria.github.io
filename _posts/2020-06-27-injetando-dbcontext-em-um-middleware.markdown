---
layout: post
title:  "Injetando DBContext em um middleware"
date:   2020-06-27 17:26:00
categories: C#
description: "Como injetar um DBContext em um middleware"
tags: c#, netcore, middleware, entityFramework, ef, InvalidOperationException, dbcontext, di, dependencyinjection
author: Luiz Faria (Luizão)
image: /assets/article_images/2020-06-27-injetando-dbcontext-em-um-middleware/background_1920.jpg
image2: /assets/article_images/2020-06-27-injetando-dbcontext-em-um-middleware/background_1920.jpg
---

Recentemente, passei por um problema ao implementar um middleware que precisava ir até a base de dados em algum momento, em uma aplicação que fazia uso do Entity Framework.

A única dependência direta do middleware era a classe de serviço com as regras de negócio, que por sua vez utilizava o repositório que fazia uso do EF. 

Toda vez que eu executava a aplicação eu recebia a seguinte mensagem de erro:

InvalidOperationException: Cannot resolve scoped service 'MyApplication.MyDbContext' from root provider.

Como resolver isso?

Para resolver precisamos entender um pouco sobre como o mecanismo de injeção de dependências do NET Core funciona, quando injetamos nossas depêndencias, normalmente, injetamos classes com regras de negócio como Transient, mas, o DBContext vai como Scoped. Mas, o que isso quer dizer? Vamos a uma breve explicação sobre como podemos injetar nossas dependências:

<ul>
<li>Transient - Uma instância é criada para cada vez que o uso da dependência se faz necessário.</li>
<li>Scoped - Uma instânia é criada por requisição.</li>
<li>Singleton - A instâcia é uma só para qualquer requisição.</li>
</ul>

Um middleware naturalmente é um singleton, e a injeção de DBContext é feita como scoped, e as classes de serviço contendo as regras de negócio normalmente são injetadas como transient... No momento de realizar a injeção da instância da classe de serviço, que é do tipo transient, no middleware o injetor simplesmente não consegue se resolver e trazer junto de si suas dependências de outro tipo, no caso scoped.

Devemos simplesmente mudar a injeção da classe de serviço para scoped?

A resposta é um categórico não.

Elas devem ser transient, devem ter uma instância para cada uso.

Então como resolvemos o problema?

Simples, o middleware recebe por parâmetro o contexto e o contexto contém o ServiceProvider com todas as injeções que foram feitas, então, precisamos simplesmente resolver a injeção no middleware no momento da execução.

Escrevi um exemplo simples do problema e da solução. Criei uma API com a boa e velha WeatherForecastController e uma estrutura de dados simples só para logar as requisições, sei que pode não ser o melhor caminho para um log de requisições, mas, o fim aqui é didático.

Para essa estrutura usei Docker e MySQL. Já falei um pouco sobre Docker e como usar o MySQL no Docker no artigo <a href="https://lfppfaria.github.io/c%23/2020/06/19/logando-queries-ef-serilog.html">"Logando queries do Entity Framework com Serilog".</a>, então, para entender um pouco mais sobre como fazer isso é só dar uma olhada lá.

A estrutura que criei foi a seguinte:

{% highlight sql %}

create schema log;

use log;

create table request(
    id int not null primary key auto_increment,
    `request_date` datetime not null,
    `request_path` varchar(500)
);

{% endhighlight %}

Depois criei uma classe simples representando essa tabela:

{% highlight c# %}
public class Log
{
    public int Id { get; set; }

    public DateTime RequestDate { get; set; }

    public string RequestPath { get; set; }
}
{% endhighlight %}

Então configurei o LogDbContext:

{% highlight c# %}
public class LogDbContext : DbContext
{
    public DbSet<Log> Logs { get; set; }

    public LogDbContext(DbContextOptions options) : base(options)
    {
    }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        //Microsoft.EntityFrameworkCore.Relational
        modelBuilder.HasDefaultSchema("log");

        //Microsoft.EntityFrameworkCore.Relational
        modelBuilder.Entity<Log>().ToTable("request");

        modelBuilder.Entity<Log>().HasKey("Id");

        //Microsoft.EntityFrameworkCore.Relational
        modelBuilder.Entity<Log>().Property("RequestDate").HasColumnName("request_date");
        modelBuilder.Entity<Log>().Property("RequestPath").HasColumnName("request_path");
    }
}
{% endhighlight %}

Escrevi uma classe simples para servir de ponte com o banco e uma classe fazendo o papel de uma classe com as regras de negócio da aplicação, só para emularmos uma situação mais próxima do mundo real.

LogRepository:

{% highlight c# %}
public class LogRepository
{
    private readonly LogDbContext _logDbContext;

    public LogRepository(LogDbContext logDbContext)
    {
        _logDbContext = logDbContext;
    }

    public async Task Insert(Log log)
    {
        _logDbContext.Logs.Add(log);

        await _logDbContext.SaveChangesAsync();
    }
}
{% endhighlight %}

LogService:

{% highlight c# %}
public class LogService
{
    private readonly LogRepository _logRepository;

    public LogService(LogRepository logRepository)
    {
        _logRepository = logRepository;
    }

    public async Task LogAsync(Log log)
    {
        await _logRepository.Insert(log);
    }
}
{% endhighlight %}

E finalmente escrevi o middleware:

{% highlight c# %}
public class LogMiddleware
{
    private readonly RequestDelegate _next;

    private LogService _logService;
    
    public LogMiddleware(RequestDelegate next, LogService logService)
    {
        _next = next;

        _logService = logService;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        var log = new Log 
        { 
            RequestDate = DateTime.Now, 
            RequestPath = context.Request.Path             
        };

        await _logService.LogAsync(log);

        // Call the next delegate/middleware in the pipeline
        await _next(context);
    }
}
{% endhighlight %}

E amarrei as coisas no Startup:

{% highlight c# %}
public void ConfigureServices(IServiceCollection services)
{
    services.AddTransient(typeof(LogRepository));
    services.AddTransient(typeof(LogService));

    //Pomelo.EntityFrameworkCore.MySql
    services.AddDbContext<LogDbContext>(options => options.UseMySql("Server=localhost;Database=log;Uid=root;Pwd=password;"), ServiceLifetime.Scoped);

    services.AddControllers();
}

public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
{
    //...

    app.UseMiddleware<LogMiddleware>();

    //...
}
{% endhighlight %}

Notem que fiz uso do mecanismo de injeção de dependência do NET CORE normalmente, como em qualquer outra aplicação. Como trata-se de um exemplo não vi necessidade de criar interfaces para cada dependência, o resultado aqui será o mesmo.

Agora basta executarmos a aplicação. E...

<img src="/assets/article_images/2020-06-27-injetando-dbcontext-em-um-middleware/erro.JPG">

Nosso erro...

E a solução é simples, basta deixarmos de injetar o LogService e resolver internamente. O método InvokeAsync que é executado quando qualquer requisição é feita recebe por parâmetro um HttpContext, esse objeto por sua vez contém uma propriedade chamada RequestServices, que nada mais é que o ServiceProvider e carrega todas as dependências injetadas na aplicação:

{% highlight c# %}
public class LogMiddleware
{
    private readonly RequestDelegate _next;

    private LogService _logService;

    //Isso dá erro, não faça assim
    //public LogMiddleware(RequestDelegate next, LogService logService)
    //{
    //    _next = next;

    //    _logService = logService;
    //}

    //Mudamos para a injeção padrão, recebendo somente o RequestDelegate
    public LogMiddleware(RequestDelegate next)
    {
        _next = next;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        //Aqui é o pulo do gato, passamos a resolver a dependência na chamada
        //usando o GetService do ServiceProvider
        _logService = (LogService)context.RequestServices.GetService(typeof(LogService));

        var log = new Log 
        { 
        RequestDate = DateTime.Now, 
        RequestPath = context.Request.Path             
        };

        await _logService.LogAsync(log);

        // Call the next delegate/middleware in the pipeline
        await _next(context);
    }
}
{% endhighlight %}

Agora quando executarmos a aplicação:

<img src="/assets/article_images/2020-06-27-injetando-dbcontext-em-um-middleware/sucesso.JPG">

E olhando os registros da tabela request no MySQL:

<img src="/assets/article_images/2020-06-27-injetando-dbcontext-em-um-middleware/sucesso-mysql.JPG">

Todo o exemplo está no meu <a href="https://github.com/lfppfaria/MiddlewareInvalidOperationException">git</a>

Espero que esse artigo ajude de alguma forma e até a próxima!