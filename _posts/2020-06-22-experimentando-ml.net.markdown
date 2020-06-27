---
layout: post
title:  "Experimentando ML.NET"
date:   2020-06-22 23:26:00
categories: C#
description: "Primeiros passos e impressões com o ML.NET"
tags: c#, netcore, machinelearning, machine learning, ml, net, ml.net
author: Luiz Faria (Luizão)
image: /assets/article_images/2020-06-22-experimentando-ml.net/background_1920.jpg
image2: /assets/article_images/2020-06-22-experimentando-ml.net/background_1920.jpg
---
Machine Learning... Já tinha ouvido falar, mas, nunca pesquisei ou busquei ler sobre o tema, então, no final do ano passado tive a oportunidade de assistir uma breve palestra sobre machine learning e fiquei bastante interessado. Posteriormente, participei do Microsoft Ignite e tive a oportunidade de acompanhar mais algumas palestras sobre o tema e isso despertou ainda mais meu interesse. 

Em todas essas palestras vi exemplos com um framework de machine learning da Microsoft, o ML.NET, e desde então vinha aguardando ter algum tempo para poder ler mais a respeito e poder implementar alguns exemplos, afim de ver o framework em funcionamento e aprender mais sobre o tema.

<h3>Machine Learning</h3>

Em uma tradução literal significa simplesmente "aprendizado de máquina".

Mas, o que isso quer dizer na prática?

Na prática isso significa que podemos ensinar uma máquina a realizar uma tarefa ou ação de forma autônoma com base em sua experiência.

E, como fazemos uma máquina ganhar experiência para poder realizar uma ação ou tomar alguma decisão?

Através de algoritmos que identificam padrões em conjuntos de dados e que são construídos sob a ótica de um determinado problema. São esses algoritmos que usamos para ensinar a máquina a agir de acordo com sua análise. Existem algoritmos para identificação de objetos em imagens, detecção de fraudes, previsões de preços ou estoque e vários outros tipos.

<h3>Diferença entre inteligência artificial e machine learning</h3>

Constantemente machine learning e inteligência artificial são confundidos, mas, para simplificar, inteligência artificial é um ramo da computação que busca treinar computadores para realizar ações que normalmente necessitariam de inteligência humana para serem realizadas, já o machine learning é um ramo da inteligência artificial que ensina os computadores a encontrar padrões em conjuntos de dados para que possa realizar ações com base nelas.

<h3>ML.NET</h3>

O ML.NET é um framework da Microsoft que nasceu com o objetivo de levar o machine learning para aplicações .NET. Ele permite o treinamento e construção de modelos usando C# ou F# e uma vez que esses modelos estejam prontos podem ser facilmente integrados a uma aplicação. 

Muito mais sobre o ML.NET pode ser visto <a href="https://docs.microsoft.com/en-us/dotnet/machine-learning/">aqui</a>.

<h3>Ciclo de vida do machine learning</h3>

O ciclo do vida do machine learning é um processo cíclico que define os passos que devem ser seguidos para extrair o melhor do machine learning e agregar valor ao negócio da companhia. A maioria das literaturas fala que esse ciclo é constituído por 5 a 7 passos, porém, vou me ater ao modelo que a Microsoft usa em suas apresentações que possui 5 passos: 

<ul>
<li>Faça a pergunta correta - Para solucionar um problema precisamos entender completamente com o que estamos lidando. Isso influenciará na preparação dos dados e na escolha do algoritmo.</li>
<li>Prepare os dados - Em um cenário realista provavelmente o analista precisará buscar informações de várias bases de dados e sistemas para conseguir formar uma massa de dados consistente. Esse etapa é de grande importância, pois, é essa massa de dados que será utilizada no treinamento do modelo.</li>
<li>Escolha o algoritmo (modelo) - O machine learning possui uma série de algoritmos para resolver vários problemas, e, por vezes, mais de um algoritmo pode resolver o mesmo tipo de problema. Nessa etapa, pode ser necessário realizar testes com mais de um para determinar qual algoritmo é melhor para solucionar seu problema.</li>
<li>Treine o modelo - Com os dados preparados e o modelo selecionado, é hora do treinamento. Aqui os dados são inseridos no modelo para que ele possa "aprender" com os dados que já temos.</li>
<li>Teste o modelo - Agora com o modelo já treinado basta testar e analisar seus resultados. Esse pode ser o momento de considerar o teste de outro algoritmo ou refinar seus dados caso os resultados sejam muito discrepantes.</li>
</ul>

<h3>Implementação</h3>

Para meu cenário parti da seguinte questão: Com base nas minhas vendas dos últimos anos, quanto vou vender nos próximos 12 meses?

Como estou realizando testes e aprendendo a usar o ML.NET em um cenário controlado, a preparação de dados não existe, simplesmente criei uma massa de dados em um formato que possa ser facilmente utilizada para treinamento do modelo.

Para isso usei MySQL rodando no Docker. Já falei um pouco sobre Docker e como usar o MySQL no Docker no artigo <a href="https://lfppfaria.github.io/c%23/2020/06/19/logando-queries-ef-serilog.html">"Logando queries do Entity Framework com Serilog".</a>, então, se quiser entender com fazer isso basta dar uma lida lá.

Seguindo, simplesmente criei um schema para receber o valor mensal total das vendas, o mês e ano da venda.

{% highlight sql %}

create schema machine_learning;

use machine_learning;

create table input_data(
    id int primary key auto_increment,
    sell_year int not null,
    sell_month int not null,
    `value` numeric(10, 2)
);

{% endhighlight %}

E criei uma procedure para popular essa tabela com valores aleatórios, dado um período:

{% highlight sql %}

USE machine_learning;

DELIMITER //

CREATE PROCEDURE populate_ml_database(begin_year int, end_year int)
BEGIN

DECLARE current_year INT;
DECLARE current_month INT;
DECLARE counter INT;

set current_year = begin_year;

year_loop: LOOP	
    SET current_month = 1;

    month_loop: LOOP
        INSERT INTO input_data VALUES(null, current_year, current_month, RAND() * 100000);        

        IF(current_month = 12) THEN
            LEAVE month_loop;
        END IF;

        SET current_month = current_month + 1;
    END LOOP month_loop;    

    IF(current_year = end_year) THEN
        LEAVE year_loop;
    END IF;

    SET current_year = current_year + 1;
END LOOP year_loop;

END //

{% endhighlight %}

Resolvi criar uma massa de três anos, 2017 a 2019. Basta executar a procedure para popularmos a base:

{% highlight sql %}

call populate_ml_database(2017, 2019);

{% endhighlight %}

Com a base de dados pronta, podemos partir pro código .NET. Pra começar criei uma aplicação do tipo console e adicionei os pacotes NuGet que vamos precisar:

<ul>
<li>MySql.Data - Será necessário para a conexão com a base MySQL.</li>
<li>Microsoft.ML - Carrega o coração do framework de machine learning.</li>
<li>Microsoft.ML.TimeSeries - Contém o algoritmo que usaremos para nosso estudo.</li>
</ul>

Com a instalação dos pacotes necessários feita podemos começar a codificar. Eu montei meu código dentro do Main mesmo.

O primeiro passo é criar um MLContext, ele é o ponto de entrada de todas as operações com o ML.NET bem como é usado para a criação e uso dos modelos.

{% highlight c# %}

var context = new MLContext();

{% endhighlight %}

Com o contexto criado, vamos carregar nossos dados, primeiramente, criei um objeto para receber os dados:

{% highlight c# %}

public class InputModel
{
    public float SellMonth { get; set; }

    public float SellYear { get; set; }

    public float Value { get; set; }
}

{% endhighlight %}

Com esse objeto criado, basta carregar os dados para o contexto. Note que na query que é executada trazemos as colunas com exatamente o mesmo nome das propriedades do objeto:

{% highlight c# %}
var dataLoader = context.Data.CreateDatabaseLoader<InputModel>();

var connectionString = "Server=localhost;Database=machine_learning;Uid=root;Pwd=password;";

var query = @"
    SELECT 
        sell_year as SellYear, 
        sell_month as SellMonth, 
        value as Value
    FROM 
        input_data;";

var dataSource = new DatabaseSource(MySqlClientFactory.Instance, connectionString, query);

var dataView = dataLoader.Load(dataSource);
{% endhighlight %}

Agora que já temos os dados carregados em nosso contexto podemos treinar o modelo, nessa etapa é de grande importância ficar atento aos valores que estamos fornecendo aos parâmetros de treino, pois, esses parâmetros contém coisas como o windowSize, seriesLength e trainSize que se utilizados de forma equivocada podem gerar resultados completamente inesperados. Como quero fazer uma previsão mais linear ao longo de um período de tempo, optei por usar o algoritmo SSA que combina elementos comuns de análise ao longo de um período de tempo com análises estatísticas e decomposições. Existem outros algoritmos que poderiam ajudar a resolver esse mesmo problema mas vou optar por implementar esse primeiro.

{% highlight c# %}
var dataPoints = context.Data.CreateEnumerable<InputModel>(dataView, false).Count(); //Quantidade de itens que tenho para dar entrada no treinamento e na previsão

var forecastPipeline = context.Forecasting.ForecastBySsa(
    outputColumnName: "ForecastedSells", //Propriedade do objeto de saída que receberá o valor previsto
    inputColumnName: "Value", //Propriedade da fonte de dados que terá seu valor previsto
    windowSize: 12, //Período do ciclo que estamos medindo, como estou medindo vendas por mês do ano, 12
    seriesLength: dataPoints, //Quantidade de entradas que tenho para fazer o previsão
    trainSize: dataPoints, //Quantidade de entradas que tenho para treinar meu modelo
    horizon: 12, //Período que prever, nesse caso os próximos 12 meses
    confidenceLevel: 0.95f, //Nível de confiança na previsão de melhor e pior cenário, quanto mais alto maior o intervalo entre as previsões de melhor e pior cenário, porém, mais confiável
    confidenceLowerBoundColumn: "LowerBoundSells", //Propriedade do objeto de saída que receberá o valor de previsão de pior cenário 
    confidenceUpperBoundColumn: "UpperBoundSells"); //Propriedade do objeto de saída que receberá o valor de previsão de melhor cenário
{% endhighlight %}

Um pouco sobre cada um desses parâmetros:

<ul>
<li>outputColumnName - Nome da propriedade no objeto de saída que será preenchida com as previsões.</li>
<li>inputColumnName - Nome da propriedade do objeto de entrada que contém os dados sobre os quais a previsão será feita.</li>
<li>windowSize - É o parâmetro mais importante. Esse parâmetro é usado para definir a janela de tempo que deverá ser utilizada pelo algoritmo para decompor os dados. Normalmente, usa-se com a maior janela de tempo possível e que faça sentido para seu negócio/cenário. Por exemplo, se seu negócio possui um ciclo com períodos semanais ou mensais (7 ou 30 dias) e a coleta de dados é feita diariamente, a janela de tempo deve ser 30 para representar a maior janela possível no seu ciclo de negócio. No nosso caso, nossa base de dados está em meses e com uma coleta de dados mensal, não em dias, portando usaremos o tamanho 12.</li>
<li>seriesLength - Quantidade de itens que será usada para realizar a previsão.</li>
<li>trainSize - Quantidade de itens que será usada para realizar o treino do modelo.</li>
<li>horizon - Indica o número de períodos para realizar a previsão. No nosso caso queremos 12 meses, usaremos 12.</li>
<li>confidenceLevel - Indica o nível de confiança da previsão, o valor deve estar entre 0 e 1 e quanto maior o valor maior será a diferença entre os limites inferiores e superiores.</li>
<li>confidenceLowerBoundColumn - Propriedade do objeto de saída que receberá o valor de previsão de pior cenário.</li>
<li>confidenceUpperBoundColumn - Propriedade do objeto de saída que receberá o valor de previsão de melhor cenário</li>
</ul>

Finalmente podemos treinar o modelo:

{% highlight c# %}
var forecaster = forecastPipeline.Fit(dataView);
{% endhighlight %}

Com o modelo devidamente treinado podemos agora criar o mecanismo de previsão. Mas, primeiro, precisamos de um objeto para receber os dados das previsões. Para isso criei a classe OutputModel:

{% highlight c# %}
public class OutputModel
{
    public float[] ForecastedSells { get; set; }

    public float[] LowerBoundSells { get; set; }

    public float[] UpperBoundSells { get; set; }
}
{% endhighlight %}

E agora criamos o mecanismo:

{% highlight c# %}
var forecastEngine = forecaster.CreateTimeSeriesEngine<InputModel, OutputModel>(context);
{% endhighlight %}

Com o mecanismo criado, podemos gravar seu modelo para que seja utilizado em outra aplicação:

{% highlight c# %}
//outputModelPath é o caminho onde o modelo será armazenado
forecastEngine.CheckPoint(context, outputModelPath); 
{% endhighlight %}

E para utilizar o modelo armazenado basta fazer dessa forma:

{% highlight c# %}
ITransformer forecaster;
//outputModelPath é o caminho onde o modelo está armazenado
using (var file = File.OpenRead(outputModelPath))
{
    forecaster = mlContext.Model.Load(file, out DataViewSchema schema);
}
{% endhighlight %}

Lembrando que esses passos de armazenar um modelo e carregar um modelo pré existente são opcionais.

Agora vamos testar nosso modelo, para isso criei um outro método, pois, achei que tudo junto no Main ficaria muito confuso:

{% highlight c# %}
static void Forecast(
    IDataView data, 
    int horizon, 
    TimeSeriesPredictionEngine<InputModel, OutputModel> forecaster, 
    MLContext context)
{
    var originalData = context.Data.CreateEnumerable<InputModel>(data, false);

    //Esse é o momento em que a previsão é feita
    var forecast = forecaster.Predict();

    //Gerando uma saída comparando os dados anteriores com a previsão
    var forecastOutput = context
        .Data
        .CreateEnumerable<InputModel>(data, false)
        .Take(horizon)
        .Select((InputModel sell, int index) =>
        {
            var pastYearsSells = originalData
                .Where(item => item.SellMonth.Equals(index + 1)).OrderBy(item => item.SellYear);

            var stringfiedPastYearsSells = pastYearsSells.Select(item => $"\t{item.SellYear} sells: {item.Value}\n");

            var month = sell.SellMonth;
            float actualSells = sell.Value;
            float lowerEstimate = Math.Max(0, forecast.LowerBoundSells[index]);
            float estimate = forecast.ForecastedSells[index];
            float upperEstimate = forecast.UpperBoundSells[index];

            return $"Month: {month}\n" +
            $"Other Year Sells:\n{string.Concat(stringfiedPastYearsSells)}\n" +
            $"Lower Estimate: {lowerEstimate}\n" +
            $"Forecast: {estimate}\n" +
            $"Upper Estimate: {upperEstimate}\n";
        });

    foreach (var item in forecastOutput)
    {
        Console.WriteLine(item);
    }
}
{% endhighlight %}

Basicamente, o que fiz nesse método foi realizar a previsão usando o método Predict e gerar um comparativo no console mostrando como foram as vendas nos anos anteriores e qual é a previsão para os próximos 12 meses. Lembrando que gerei uma base aleatória de dados, portanto, para cada nova carga da base outras previsões serão geradas.

{% highlight terminal %}
Month: 1
Other Year Sells:
        2017 sells: 1193,96
        2018 sells: 33107,31
        2019 sells: 90057,62

Lower Estimate: 33906,023
Forecast: 84936,03
Upper Estimate: 135966,03

Month: 2
Other Year Sells:
        2017 sells: 41002,29
        2018 sells: 7228,41
        2019 sells: 30787,47

Lower Estimate: 0
Forecast: 32360,107
Upper Estimate: 83615,33

Month: 3
Other Year Sells:
        2017 sells: 1429,56
        2018 sells: 36820,15
        2019 sells: 83764,47

Lower Estimate: 35111,83
Forecast: 86367,336
Upper Estimate: 137622,84

Month: 4
Other Year Sells:
        2017 sells: 84140,95
        2018 sells: 62415,49
        2019 sells: 26459,97

Lower Estimate: 0
Forecast: 32327,967
Upper Estimate: 83843,89

Month: 5
Other Year Sells:
        2017 sells: 16416,07
        2018 sells: 1617,02
        2019 sells: 81006,45

Lower Estimate: 36270,27
Forecast: 87786,41
Upper Estimate: 139302,55

Month: 6
Other Year Sells:
        2017 sells: 29657,51
        2018 sells: 20838,64
        2019 sells: 25652,35

Lower Estimate: 0
Forecast: 31779,867
Upper Estimate: 83675,74

Month: 7
Other Year Sells:
        2017 sells: 99039,31
        2018 sells: 99342,15
        2019 sells: 85242,4

Lower Estimate: 37014,18
Forecast: 88913,836
Upper Estimate: 140813,48

Month: 8
Other Year Sells:
        2017 sells: 6224,06
        2018 sells: 34194,81
        2019 sells: 49254,95

Lower Estimate: 0
Forecast: 31814,707
Upper Estimate: 84291,22

Month: 9
Other Year Sells:
        2017 sells: 34002,36
        2018 sells: 72947,63
        2019 sells: 90547,54

Lower Estimate: 36870,52
Forecast: 89349,805
Upper Estimate: 141829,1

Month: 10
Other Year Sells:
        2017 sells: 51339,61
        2018 sells: 62153,69
        2019 sells: 4972,86

Lower Estimate: 0
Forecast: 32675,904
Upper Estimate: 85796,37

Month: 11
Other Year Sells:
        2017 sells: 54690,97
        2018 sells: 91925,59
        2019 sells: 53221,68

Lower Estimate: 37100,445
Forecast: 90221,08
Upper Estimate: 143341,72

Month: 12
Other Year Sells:
        2017 sells: 19436,05
        2018 sells: 73166,88
        2019 sells: 51189,83

Lower Estimate: 0
Forecast: 33130,65
Upper Estimate: 86683,63
{% endhighlight %}

Todo o código está disponível no meu <a href="https://github.com/lfppfaria/POCML.NET">git</a>.

<h3>Conclusão</h3>

O machine learning já é uma realidade há algum tempo e está cada vez mais presente nas empresas e em nosso cotidiano (ainda que a gente muitas vezes nem faça ideia disso) e o ML.NET fornece meios de trazê-lo ao universo Microsoft de forma acessível. Ele foi construído com os desenvolvedores e aplicações .NET em mente, permitindo que façamos uso de todo conhecimento que já temos aliado às bibliotecas que já conhecemos para que possamos integrar o machine learning em nossas aplicações de forma rápida.

Espero que tenham gostado e muito porvavelmente voltarei a falar do assunto por aqui, já que ainda há muito o que ser explorado.