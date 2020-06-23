---
layout: post
title:  "Fluent Validation"
date:   2020-06-14 15:18:00
categories: C#
tags: C#, netcore, validator, fluentvalidation
image: /assets/article_images/2020-06-13-bem-vindos/landscape.jpg
image2: /assets/article_images/2020-06-13-bem-vindos/landscape.jpg
---

Hoje, estudando e lendo alguns artigos, me deparei com uma biblioteca chamada <a href="https://fluentvalidation.net">Fluent Validation</a> que ajuda a resolver um problema recorrente em Web APIs: a validação.

Todos sabemos que, além de uma boa prática, é essencial para o negócio validar os dados que estão chegando em uma requisição e, quanto mais validações e regras, a coisa pode acabar virando um verdadeiro pesadelo.

Mas, vamos ao nosso problema e a como o Fluent Validation se propõe a resolvê-lo.

<h3>O problema</h3>
<br />
Normalmente quando estamos falando de validação das propriedades de uma classe acabamos nos rendendo ao atributo "Required" ou acabamos fazendo tudo manualmente. Quando usamos o RequiredAttribute acabamos "sujando" nossa classe e acabamos tendo algo mais ou menos assim:

{% highlight csharp %}
public class Person
{
    [Required]
    public string Name { get; set; }

    [Required]
    public string Email { get; set; }

    public string PhoneNumber { get; set; }

    public string Address { get; set; }

    [Required]
    public DateTime BirthDate { get; set; }
}
{% endhighlight %}

Além disso, o RequiredAttribute só vai validar se o preenchimento da propriedade é obrigatório.

"Mas, Luizão, existem outros atributos pra validar outras coisas". 

Sim, temos outros atributos para validar outros pontos das propriedades da nossa classe e da mesma forma que o RequiredAttribute polui o código e é limitado, os outros atributos também são. Fatalmente acabamos fazendo alguma validação manual e acabamos com um cenário mais indesejado ainda: a lógica de validação acaba ficando espalhada.

Eu, particularmente, prefiro não ter essa lógica espalhada, mas aí acabamos escrevendo muito código para realizar nossas validações e pior, muitas vezes acabamos duplicando código. Nesse momento entra o Fluent Validation.

<h3>A solução</h3>
<br />
Para começar vamos criar um projeto de Web API com net core. Aqui usei o 3.0.

Em seguida vamos adicionar a dependência do FluentValidation.AspNetCore:

<img src="/assets/article_images/2020-06-14-fluent-validation/nuget.JPG">

O Fluent Validation já possui uma série de regras para facilitar a validação das propriedades das nossas classes. Vamos criar uma classe de validação com as regras que desejo para a minha classe "Person".
Sei que parece ter muita coisa, mas, vou explicar o que cada método está fazendo:

{% highlight csharp %}
public class PersonValidator : AbstractValidator<Person>
{
    public PersonValidator()
    {
        RuleFor(person => person.Name)
            .NotEmpty().WithMessage("Nome não pode ser vazio.")
            .Must(IsValidName).WithMessage("Nome deve ser composto somente de letras.");

        RuleFor(person => person.Email)
            .NotEmpty().WithMessage("Email não pode ser vazio.")
            .EmailAddress().WithMessage("Endereço de email informado não é um endereço válido.");

        RuleFor(person => person.BirthDate)
            .NotEmpty().WithMessage("Data de nascimento deve ser informada.")
            .LessThan(DateTime.Now).WithMessage("Data de nascimento não pode ser superior a data atual");
    }

    private bool IsValidName(string name)
    {
        if (string.IsNullOrEmpty(name))
            return true;  

        return !name.All(Char.IsDigit);
    }
}
{% endhighlight %}

Note que a classe deve herdar de AbstractValidator<T> onde T é o tipo da classe que desejo validar, no nosso caso "Person".

Voltando à nossa classe "Person", agora podemos remover os RequiredAttribute e teremos ela limpa:

{% highlight csharp %}
public class Person
{
    public string Name { get; set; }

    public string Email { get; set; }

    public string PhoneNumber { get; set; }

    public string Address { get; set; }

    public DateTime BirthDate { get; set; }
}
{% endhighlight %}

Então, criei uma controller para receber uma instância da nossa classe "Person", via POST:

{% highlight csharp %}
[HttpPost]
public async Task<IActionResult> Create(Person person)
{
    return Ok();
}
{% endhighlight %}

E na nossa Startup.cs simplesmente configurei para que as controllers do nosso projeto registrem as classes de validação.

{% highlight csharp %}
public void ConfigureServices(IServiceCollection services)
{
    services.AddControllers().AddFluentValidation(configuration =>
    {
        //Dizendo ao Fluent Validation para registrar todas as classes de validação no assembly
        configuration.RegisterValidatorsFromAssemblyContaining<Startup>();
        //Dizendo ao Fluent Validation não executar mais a validação do net core
        configuration.RunDefaultMvcValidationAfterFluentValidationExecutes = false;
    });
}
{% endhighlight %}

Agora, quando consumirmos o POST, via Postman, teremos os seguintes resultados:

Um objeto devidamente preenchido nos dará um status code 200:

<img src="/assets/article_images/2020-06-14-fluent-validation/200.JPG">

Agora, se eu passar um json sem a propriedade "Name" preenchida terei o seguinte retorno:

<img src="/assets/article_images/2020-06-14-fluent-validation/nameNullOrEmpty.JPG">

Recebemos como retorno um status code 400 e uma mensagem "Nome não pode ser vazio." exatamente conforme configurada na nossa classe de validação. 

Notem que não fiz nada além de configurar minhas validações na classe de validação do objeto "Person" e configurar no Startup.cs.

Agora, vamos a mais um exemplo. 

Vou tentar inserir um endereço de email inválido:

<img src="/assets/article_images/2020-06-14-fluent-validation/invalidEmailAddress.JPG">

Notem que enviei um endereço sem o '@' e com um '.' a menos.

O Fluent Validation já tem um método para validação específica de endereços de email e não precisei fazer mais nada para usar isso além da configuração:

{% highlight csharp %}
RuleFor(person => person.Email)
    .NotEmpty().WithMessage("Email não pode ser vazio.")
    .EmailAddress().WithMessage("Endereço de email informado não é um endereço válido.");
{% endhighlight %}

Mais um cenário que o Fluent Validation nos oferece é a possibilidade de consumir um método próprio para alguma validação mais específica.

Vou enviar um objeto com um nome composto somente de números:

<img src="/assets/article_images/2020-06-14-fluent-validation/invalidName.JPG">

Voltando ao código na nossa classe de validação, notamos que a regra de validação da propriedade "Name" consome a função "Must", que recebe como parâmetro uma function que por sua vez recebe como parâmetro um objeto do mesmo tipo da propriedade que está sendo validada e, então, retorna um bool que simplesmente indicará se a validação ocorreu com sucesso ou não.

{% highlight csharp %}
RuleFor(person => person.Name)
    .NotEmpty().WithMessage("Nome não pode ser vazio.")
    .Must(IsValidName).WithMessage("Nome deve ser composto somente de letras.");
{% endhighlight %}

E a função que o "Must" está consumindo:

{% highlight csharp %}
private bool IsValidName(string name)
{
    if (string.IsNullOrEmpty(name))
        return true;

    return !name.All(Char.IsDigit);
}
{% endhighlight %}

Com isso podemos usar todas as funções de validação que o Fluent Validation nos fornece e caso o desenvolvedor venha a precisar de alguma regra mais específica é muito simples escrever essa regra e trazê-la ao universo do Fluent Validation.

<h3>Validando manualmente</h3>

Em algumas circunstâncias vamos precisar disparar essas validações fora da controller, essa validação automática amarrada na controller é uma facilidade que o Fluent Validation nos fornece, mas, não é a única forma de usar.

Para consumir as validações manualmente basta fazer dessa forma:

{% highlight csharp %}
[HttpPost]
public async Task<IActionResult> Create(Person person)
{
    //Instanciando o PersonValidator
    var validator = new PersonValidator();

    //Executando o método de validação
    var validationResult = validator.Validate(person);

    //Verificando o resultado da validação do objeto
    if (!validationResult.IsValid)
        return BadRequest(validationResult.Errors);

    return Ok();
}
{% endhighlight %}

<h3>Conclusão</h3>

Validação é o tipo de problema que em todo projeto acaba parando a equipe, ainda que por alguns minutos, para decidir como será atacado ou resolvido, afinal, cada projeto tem suas demandas e particularidades e não existe uma única solução capaz de resolver todos os cenários. 

O Fluent Validation traz uma nova perspectiva sobre como podemos resolver esse tipo de situação.

Vou deixar o projeto que usei para teste no <a href="https://github.com/lfppfaria/POCFluentValidation">git</a>.

Espero que esse artigo seja útil para alguém que esteja pensando em usar o Fluent Validation ou que esteja passando por problemas em suas validações e queira uma nova luz sobre o assunto.

Abraços e até a próxima!