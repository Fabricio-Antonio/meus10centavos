---
title: "Git prático para iniciantes - O início de um projeto"
slug: git-prático-para-iniciantes
date: 2026-04-02
tags: versioning, guide
author: Matheus Henrique
---

# Introdução

Antes de começar, já devo avisar que o objetivo desta série não é ensinar git de A até Z como muitos fazem, pois eu não acho que conseguiria fazer algo melhor do que a [documentação do git](https://git-scm.com/book/pt-br/v2), que tem uma leitura fácil e entendível para quem está iniciando, além de ter uma boa parte já traduzida para o português brasileiro, caso tenha dificuldades com o inglês.

Meu objetivo é simplificar seu aprendizado, abordando o essencial, destrinchando as entrelinhas e explicando de uma forma fácil para iniciantes.

> **Nota:** Entenda que estamos falando do git (ferramenta de controle de versão), não do GitHub (repositório remoto). Isso porque em algum momento você irá trabalhar em empresas que utilizam outros, como BitBucket e GitLab. Pesquisem por conta sobre esses repositórios remotos. Se em algum momento eu me referir a repositório remoto, assuma que estou falando do GitHub para esses exemplos, mas o mesmo se aplicaria de forma similar aos outros citados.

# Inicializando um repositório na sua máquina

Vamos supor que você já assistiu seu curso da Udemy ou do YouTube e já sabe o básico do funcionamento de git. Logo, caso tenha entendido corretamente, o primeiro passo é inicializar um repositório dentro do diretório do seu projeto com o comando `git init`.

```
enzo@pc-do-enzo:~/projeto$ git init
Initialized empty Git repository in /home/enzo/projeto/.git/
enzo@pc-do-enzo:~/projeto$
```

O que você precisa entender agora é que, ao inicializar um repositório, um diretório chamado `.git` será criado no diretório do projeto e que por padrão, nenhum dos seus arquivos estará, mas o git já consegue vê-los. Além disso, todos seus arquivos começarão com o estado `untracked` ou "não rastreado".

```
enzo@pc-do-enzo:~/projeto$ git status
On branch main

No commits yet

Untracked files:
  (use "git add <file>..." to include in what will be committed)
        index.html
        style.css

nothing added to commit but untracked files present (use "git add" to track)
enzo@pc-do-enzo:~/projeto$
```

# As três áreas do git

É essencial que você entenda que o git possui três áreas e que seus arquivos vão ter estados nessas áreas. As áreas basicamente são: _working directory_, _staging area_ e o diretório `.git` do projeto.

![As três áreas do git](../content/media/git-areas.png)

Resumindo o fluxo nessas áreas de maneira simples.

- Você faz alterações no seu projeto no _working directory_
- Você adiciona as alterações na _staging area_
- Você registra as alterações, que irão ser armazenadas no diretório `.git`

# Preparando para o commit

Você, como um ótimo desenvolvedor, terminou a primeira tarefa do projeto, que era criar o código HTML e o CSS de uma página importante e agora você quer que o git faça o versionamento desse código.

Primeiro, precisamos adicionar as alterações que fizemos para a área de staging, ou "área de preparo". Para isso, utilizamos o comando `git add` que pode ser chamado das seguintes formas:

- `git add index.html`: Adiciona um arquivo específico
- `git add index.html style.css`: Adiciona múltiplos arquivos
- `git add foo/bar`: Adiciona todos os arquivos de um diretório
- `git add .`: Adiciona todos os arquivos

```
enzo@pc-pc-do-enzo:~/projeto$ git add index.html style.css
enzo@pc-pc-do-enzo:~/projeto$ git status
On branch main

No commits yet

Changes to be committed:
  (use "git rm --cached <file>..." to unstage)
        new file:   index.html
        new file:   style.css

enzo@pc-pc-do-enzo:~/projeto$
```

Como pode ver, as informações do comando `git status` são bem claras:

- **On branch main**: Estamos na ramificação principal
- **No commits yet**: Ainda não há nenhum commit no repositório
- **Changes to be committed**: Mudanças que serão "commitadas". Preste atenção nessa parte, pois os items da lista abaixo, irão para o próximo commit. Caso você queira retirar um arquivo que você colocou por engano, basta executar `git rm --cached <nome-arquivo>` para retirar da _staging area_ de volta para o _working directory_.

Agora, eu realmente espero que você tenha se questionado do seguinte:

> "E se eu fizer uma modificação em um arquivo que eu já adicionei na _staging area_?"

E aqui entramos em outro conceito importante que não se aplica somente ao git, mas em muitas ferramentas que você vai encontrar e ele se chama _**diffing**_.

Em termos simples, _**diffing**_ é o processo de comparação entre arquivos, para saber o que mudou de um arquivo para outro. Você pode pensar nisso como um "jogo dos 7 erros", onde você tinha duas imagens e precisava encontrar a diferença entre elas.

Graças ao _**diffing**_, o git consegue te dizer exatamente quais linhas foram adicionadas (+) e quais linhas foram removidas (-) na comparação de uma versão anterior de um arquivo a uma versão atual.

```
enzo@pc-do-enzo:~/projeto$ git diff
diff --git a/index.html b/index.html
index c3939a0..e60d682 100644
--- a/index.html
+++ b/index.html
@@ -7,12 +7,13 @@
     <link rel="stylesheet" href="style.css">
   </head>
   <body>
-    <h1>Página inicial</h1>
+    <h1>Página inicial alterada</h1>
     <p>
       Lorem ipsum dolor, sit amet consectetur adipisicing elit. Harum ipsum
       temporibus minima explicabo molestiae repellendus deleniti dolore, at
       vitae illo numquam eaque veritatis exercitationem. Aliquam repudiandae
       similique quaerat quia optio.
     </p>
+    <p>Uma nova alteração</p>
   </body>
 </html>
enzo@pc-do-enzo:~/projeto$
```

Então, quando você realiza a modificação de um arquivo que já está na _**staging area**_, esse arquivo terá uma versão na _**staging area**_ que irá para o próximo _commit_ e uma versão em estado _**modified**_ no _**working directory**_.

```
enzo@pc-do-enzo:~/projeto$ git status
On branch main

No commits yet

Changes to be committed:
  (use "git rm --cached <file>..." to unstage)
        new file:   index.html
        new file:   style.css

Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git restore <file>..." to discard changes in working directory)
        modified:   index.html

enzo@pc-do-enzo:~/projeto$
```

> "Mas e se quiser adicionar a nova alteração que fiz junto a que já está na _**staging area**_?"

Basta adicionar essa alteração com o `git add` quantas vezes forem necessárias.

# Realizando o commit

A partir deste ponto, vamos assumir que você já fez e adicionou tudo o que era necessário e está pronto para o commit.

> Mas o que é esse tal de commit que tanto é falado?

Um _**commit**_ é um objeto do git que de fato vai registrar uma versão de código dentro do seu repositório git. Ele trás metadados como:

- O autor do _commit_
- A data em que ele foi realizado
- Uma descrição
- O _hash_ do _commit_ (isso é o principal para o git, pois podemos rastrear a qualquer momento esse _commit_ e ver o que foi feito nele)
- Quem aprovou seu _merge_ em alguma ramificação (mais a frente você verá o conceito de _merge_, ou mesclagem)

Para realizar o commit, você deve executar o comando `git commit`, que pode receber algumas opções, mas por enquanto vamos falar somente da _flag_ `-m`

Se você deseja realizar o _commit_ com uma descrição simples:

`git commit -m 'feat: implementação da página inicial do projeto'`

Se você deseja realizar o commit com uma descrição simples e informar mais detalhes:
`git commit`

Isso irá abrir o editor padrão configurado para o git, para que você possa escrever mais detalhes sobre o _commit_. Você terá instruções em forma de comentários, indicando como você deve estruturar a mensagem.

```
GNU nano 7.2                                 /home/enzo/projeto/.git/COMMIT_EDITMSG *

feat: implementação da página inicial do projeto
<ENTER>
- Criação do header fixo
- Criação de efeito parallax
- Carregamento de fontes via CDN

# Please enter the commit message for your changes. Lines starting
# with '#' will be ignored, and an empty message aborts the commit.
#
# On branch main
#
# Initial commit
#
# Changes to be committed:
#       new file:   index.html
#       new file:   style.css
#
# Changes not staged for commit:
#       modified:   index.html
#
```

Ao salvar e sair, seu _commit_ terá sido realizado.

```
enzo@pc-pc-do-enzo:~/projeto$ git commit
[main (root-commit) af17a38] feat: implementação da página inicial do projeto
 2 files changed, 32 insertions(+)
 create mode 100644 index.html
 create mode 100644 style.css
enzo@pc-pc-do-enzo:~/projeto$
```

E você pode ver detalhes sobre esse _commit_ e outros, obtendo seus logs, com o comando `git log`.

```
commit af17a38bca106c5316239f9e209cc3c1d43fd5c5 (HEAD -> main)
Author: Enzo da Silva <enzosilva@gmail.com>
Date:   Fri Feb 6 03:42:34 2026 -0300

    feat: implementação da página inicial do projeto

    - Criação do header fixo
    - Criação de efeito parallax
    - Carregamento de fontes via CDN
```

# Conclusão

Nesta primeira parte da série, você aprendeu a inicializar um repositório git, preparar alterações para seu \_**commit** e como realizá-lo.

Além disso, expliquei sobre o _**diffing**_, que é o processo que o git usa para fazer comparações entre arquivos de versões diferentes, para saber qual mudança houve entre eles.

Como material complementar, eu fortemente recomendo que você leia os capitulos 1.1 até o 2.3 da [documentação do git](https://git-scm.com/book/pt-br/v2) para entender com mais detalhes o que foi abordado aqui, pois sempre utilizo ele como base quando preciso explicar git

# Próximos passos

No próximo capítulo, pretendo falar sobre:

- Como editar a mensagem de um _commit_
- Como reverter um _commit_ que já foi realizado
- Como adicionar alterações a um _commit_ já realizado
- Repositórios remotos
- Configuração de chave SSH
- Assinatura de commits (isso aqui é importante e muita gente não sabe que existe e como fazer)

# Agradecimentos

Queria deixar registrado, meu agradecimento ao meu amigo Fabrício, que criou este espaço e permitiu que eu e outras pessoas compartilhássemos nossos conhecimentos com vocês.

# Referências

https://git-scm.com/book/en/v2

https://git-scm.com/book/pt-br/v2
