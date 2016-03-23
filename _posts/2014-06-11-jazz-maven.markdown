---
layout: post
title:  "Gerando releases usando os plugins Maven Release e JazzRTC"
date:   2014-06-11 12:00:00
comments: true
categories: maven java jazzrtc release
---

Eu não sei quanto a vocês, mas eu não gosto de procedimentos repetitivos. Considero um gigantesco desperdício de tempo fazer algo que eu sei que pode ser feito automaticamente através de software. Afinal, eu tenho um computador aqui, bem na minha frente, e quero que ele seja meu escravo, não o contrário. Por isso, considero importante gastar um pouco do tempo de desenvolvimento pesquisando formas de melhorar o processo. 

Se você utiliza o IBM Jazz RTC, saiba que há, sim, um plugin do Maven específico para ele. Para que você não perca seu tempo com os mesmos problemas que eu tive, vou compartilhar com vocês como gerar releases automaticamente, criando snapshots e baselines automaticamente. Ok, vamos lá!

# Pré-Requisitos

Para seguir as instruções, você precisará de:

* Um Eclipse com o plugin do Jazz (ALM).
* Conhecimentos básicos sobre RTC (criação de workspaces, etc).
* Conhecimentos básicos de linha de comando.
* Um Maven instalado e pronto para ser usado na linha de comando.
* JDK do Java 6 ou superior pronto para ser usado na linha de comando. Recomendo fortemente o Java 7. 

# Configurando o projeto

## Passo 1: Criar um workspace 

É uma boa prática criar um workspace específico para as builds. Dessa forma, você não confunde o seu workspace local de desenvolvimento com o workspace utilizado para as builds. Vai por mim, experiência própria... Usarei aqui o nome WS-PROJ-BUILD para o workspace de builds.

## Passo 2: Configurar os plugins release e RTC no pom.xml

Para poder gerar releases, você precisará do plugin release referenciando o plugin do Jazz RTC. Não se preocupe, é bem simples. É só adicionar dentro de build-plugins o conteúdo como abaixo:

{% highlight xml %}
<build>
    <plugins>
        <!-- Outros plugins aqui -->
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-release-plugin</artifactId>
            <version>2.4.2</version>
            <dependencies>
                <dependency>
                    <groupId>org.apache.maven.scm</groupId>
                    <artifactId>maven-scm-provider-jazz</artifactId>
                    <version>1.8.1</version>
                </dependency>
                <dependency>
                    <groupId>org.apache.maven.scm</groupId>
                    <artifactId>maven-scm-api</artifactId>
                    <version>1.8.1</version>
                </dependency>
            </dependencies>
        </plugin>
        <!-- Outros plugins aqui -->
    </plugins>
{% endhighlight %}

## Passo 3: Referenciar o repositório RTC e o workspace no pom.xml

Para que o Maven conheça qual repositório deve acessar, você precisa adicionar a tag "scm" referenciando o repositório e o workspace de build no seu arquivo pom.xml

{% highlight xml %}
<scm>
<connection>scm:jazz:usuario@https://alm.minhaempresa/ccm:WS-PROJ-BUILD</connection> 
</scm>
{% endhighlight %}

Não se preocupe com o parâmetro "username", você poderá especificar um nome de usuário ao gerar o release.

## Passo 4: Instalar o cliente RTC

O plugin do RTC, infelizmente, não funciona sozinho. Você precisa baixar o "SCM Tools" no site do Jazz Team Concert. Baixe a versão compatível com a versão do RTC no servidor. No meu caso, a versão compatível é [esta aqui](https://jazz.net/downloads/rational-team-concert/releases/4.0.2/RTC-scmTools-Linux-4.0.2.zip). Sim, você precisará criar um usuário no jazz.net, blablabla... :(

Depois de baixado o cliente RTC, descompacte-o no local de sua preferência (eu criei um diretório "apps" para guardar os aplicativos que baixo). Por exemplo, se você selecionar o diretório de destino "~/apps", será criado o diretório "~/apps/jazz". Depois de descompactado, edite o seu arquivo .bashrc do seu diretório pessoal e adicione o conteúdo abaixo no final do arquivo. 

{% highlight bash %}
PATH=$PATH:~/apps/jazz/scmtools/eclipse/bin
export PATH
{% endhighlight %}

Com isso, seu ambiente está pronto para gerar releases através do Maven!

# Gerando uma release

## Passo 1: Fazendo o login no repositório

Ok, agora que você pode gerar releases, vamos primeiro fazer o login no repositório e fazer o "cache" da autenticação, para que você não precise digitar a senha para cada pequeno comando:

{% highlight bash %}
/home/usuario$ scm login -r https://alm.minhaempresa/ccm -u usuario -c
{% endhighlight %}

A opção "-c" irá deixar gravada sua autenticação para que você não precise fazer login sempre que for executado um comando.

## Passo 2: Fazer o checkout do workspace build

Agora você não precisa baixar o workspace através do Eclipse, você pode usar a linha de comando para poupar tempo!

{% highlight bash %}
/home/usuario$ scm load WS-PROJ-BUILD -r https://alm.minhaempresa/ccm -d ws-proj-build
{% endhighlight %}

O projeto será baixado no diretório ws-proj-build.

## Passo 3: Executar o maven:release

Agora, dentro do diretório ws-proj-build, você pode gerar uma release maven. Você pode especificar o usuário aqui, mas lembre-se de que ele tem que ser o mesmo usuário com o qual você fez login anteriormente! 

{% highlight bash %}
/home/usuario$ mvn release:prepare -Dusername=usuario
{% endhighlight %}

Eis o que o plugin fará, basicamente:

* Validar se a versão do projeto termina com "-SNAPSHOT". 
* Validar se o workspace não tem alterações pendentes. A release não pode ser feita se você mudou algum arquivo depois de baixar o workspace. 
* Realizar uma build do projeto. Afinal, você não quer fazer uma release de um projeto que nem sequer compila :)
* Pedir qual a versão que você quer gerar. O padrão será a versão do pom.xml menos o "-SNAPSHOT". Por exemplo, se no pom consta "1.0-SNAPSHOT", a versão será "1.0"
* Pedir a próxima versão de desenvolvimento que será gerada. O padrão é a versão release acrescida incrementada no último dígito e acrescendata de "-SNAPSHOT". Por exemplo, se a versão release selecionada foi "1.0", a próxima versão será "1.1-SNAPSHOT".
* Mudar a versão do pom.xml para a versão release. Se for um projeto agregador, todos os projetos filhos têm seus pom.xml alterados também.
* Gerar um changeset com as alterações nos arquivos pom.xml.
* Gerar um snapshot, uma baseline e um workspace apontando para essa baseline de release.
* Mudar a versão do pom.xml para a versão de desenvolvimento. Se for um projeto agregador, todos os projetos filhos têm seus pom.xml alterados também.
* Gerar um changeset com as alterações nos arquivos pom.xml.

Por que o workspace é gerado? Porque, depois, se você quiser, pode utilizar esse workspace como a base para realizar correções em releases específicas, por exemplo.

# Conclusão

O plugin release do Maven é bastante útil para executar todos os passos necessários para gerar uma release que você pode referenciar sem problemas. Se você executar o processo manualmente, pode ter problemas se você, por exemplo, se esquecer de gerar a baseline. Lembre-se de que o RTC não permite de forma trivial que você simplesmente referencie o repositório em uma data específica. É por essas e outras que é sempre bom tentar automatizar procedimentos como a geração de builds.







