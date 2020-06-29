---
layout: post
title:  "Observabilidade em aplicações"
date:   2020-06-30 23:59:00
categories: C#
description: "Como adicionar observabilidade em aplicações"
tags: c#, netcore, observabilidade
author: Luiz Faria (Luizão)
image: /assets/article_images/2020-06-27-como-adicionar-observabilidade-em-aplicacoes/background_1920.jpg
image2: /assets/article_images/2020-06-27-como-adicionar-observabilidade-em-aplicacoes/background_1920.jpg
---

Talvez o termo seja estranho para muitos mas tenho certeza que todos já passaram por ao menos um dos principais aspectos da observabilidade em uma aplicação, o log. 

A ideia desse post não é entrar em detalhes técnicos, mas somente trazer à luz o tema e passar sobre algumas tecnologias que podem nos ajudar a trazer isso para nossas aplicações.

<h3>Um pouco sobre observabilidade</h3>

Atualmente a maioria das empresas e equipes de desenvolvimento de software trabalham focadas em entregas que possam agregar valor ao negócio mais rapidamente e de forma contínua. Claro que essas entregas passam (ou pelo menos deveriam passar) por todo um ciclo de qualidade, com testes unitários, testes de carga e testes automatizados, mas, mesmo em um cenário ideal problemas podem acontecer e é justamente aí que a observabilidade entra. 

Quando falamos de um software bem instrumentado não falamos somente de logs, claro que o log é sim muito importante, mas veja, quantos desenvolvedores você conhece que olham diariamente os logs da aplicação afim de encontrar comportamentos fora do comum? Aposto que poucos! Pois é... Infelizmente essa é a realidade, acabamos não olhando os logs como deveríamos e o log acaba relegado a uma ferramenta utilizada somente para tratar erros de forma reativa, ou seja, depois que já aconteceram e o usuário passou por eles. 

E como podemos melhorar esse cenário?

Através do uso de mais algumas ferramentas além do log e também de uma mudança de hábitos da equipe, afinal, não adianta você ter uma estrutura espetacular cuidando de seu sistema e ignorá-la não é mesmo?

Então, o que é observabilidade?

Resumidamente é a habilidade de observar como um sistema se comporta através dos dados que ele nos fornece.

Mas como o sistema nos fornece esses dados?

Vamos aos pilares da observabilidade...

<h3>Os pilares da observabilidade</h3>

<ul>
<li>Log - Responsável por coletar insformações e armazená-las para análise da equipe de desenvolvimento ou de alguma ferramenta que ajude nessa análise. Muitas vezes uma boa olhada no log nos ajuda a resolver diversos problemas, principalmente, quando esses problemas lançam exceções e nossa aplicação está preparada que capturá-las e armazená-las devidamente. Mas o log não é só log de exceções, qualquer informação que o desenvolvedor ou a equipe julgue relevante para ajudar a resolver ou prever um problema pode e deve ser armazenada. Algumas ferramentas que ajudam com log são o <a href="https://logging.apache.org/log4net/">Log4Net</a>, o <a href="https://serilog.net">Serilog</a>, o <a href="https://elmah.io">elmah</a> e o <a href="https://nlog-project.org">NLog</a>.</li>
<li>Trace - Um cenário muito comum atualmente é o de aplicações distribuídas, desenvolvidas em uma arquitetura orientada a microserviços ou simplesmente orientada a serviços. Diversas vezes temos um problema como uma lentidão ou mesmo um erro e acabamos levando algum tempo para identificar qual é o serviço que está nos causando o problema, o trace ajuda nesse tipo de situação, nos dando uma visão de detalhes como tempo médio das requisições ou requisições com erro/sucesso. Entre as ferramentas que ajudam com o tracing podemos destacar o <a href="https://zipkin.io">Zipkin</a> e o <a href="https://www.jaegertracing.io">Jaeger</a></li>.
<li>Métricas - Mais voltado ao monitoramento de recursos utilizados pela aplicação como consumo de memória e de CPU da aplicação, porém, podendo envolver outros pontos como, por exemplo, filas, se a aplicação usa filas, a contagem de mensagens paradas na fila faz parte desse pilar e deve ser observada através de sua interface de monitoramento. Nesse ponto vale destaque para o <a href="https://prometheus.io/">Prometheus</a> e o <a href="https://grafana.com/">Grafana</a> </li>
</ul>

Como já dito anteriormente, não adianta ter uma ótima estrutura e ignorá-la, portanto, além de criar o hábito de acompanhar os dados que essas ferrametas fornecem diariamente é importate ter as seguintes perguntas em mente entre suas entregas:

<ul>
<li>Estamos com mais erros que antes?</li>
<li>Estamos com novos erros?</li>
<li>Estamos mais lentos?</li>
<li>O uso de CPU e de memória aumentou?</li>
</ul>

E provavelmente podemos encaixar mais perguntas nessa lista dependendo do seu cenário, mas creio que essas sejam as principais, ou, no mínimo se encaixam em todas (ou quase todas) aplicações.

<h3>Conclusão</h3>

Observabilidade é um tema extenso e com várias tecnologias que podem nos ajudar a chegar em um patamar aceitável. Espero que esse breve artigo tenha ajudado a esclarecer a importância de conseguirmos acompanhar os eventos da aplicação e mostrar algumas ferramentas que podem ser utilizadas para esse fim. 