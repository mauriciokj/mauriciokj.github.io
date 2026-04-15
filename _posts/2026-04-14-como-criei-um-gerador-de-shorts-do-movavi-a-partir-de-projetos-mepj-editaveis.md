---
layout: default
title: "Como criamos um gerador de Shorts do Movavi a partir de projetos .mepj editáveis"
date: 2026-04-14 21:19:00 -0300
categories: [movavi, automacao, shorts, jekyll]
tags: [movavi, shorts, automacao, mepj, json, python, edicao-de-video]
---

![Gerador de Shorts do Movavi](/assets/images/posts/2026-04-14-movavi-shorts-generator/movavi-shorts-generator-blog-hero.png)

Nos últimos dias eu trabalhei em uma ideia simples, mas que acabou virando uma ferramenta bem útil: um jeito repetível de gerar projetos do Movavi para vídeos curtos, sem precisar reconstruir tudo manualmente a cada novo short.

O resultado foi um **gerador local de Shorts do Movavi** que lê um template `.mepj`, troca o áudio e as imagens, mantém o estilo visual e produz um novo projeto pronto para abrir no Movavi.

Se você quiser ver o projeto ou contribuir, o repositório está aqui:

<https://github.com/mauriciokj/shorts-project-generator>

## O que eu queria resolver

O fluxo manual era sempre o mesmo:

- abrir um projeto modelo
- trocar o áudio
- trocar as imagens
- manter o visual
- evitar quebrar a estrutura do projeto
- testar tudo de novo no Movavi

Parece simples, mas quando você tenta automatizar isso, o negócio complica.

Os arquivos de projeto do Movavi até podem ser editados, mas o programa é bastante sensível. Se você remove o clip errado, altera o campo errado ou reconstrói estrutura demais, o app pode recusar o projeto ou simplesmente ignorar mídias.

Então o desafio não era só “gerar um arquivo”. O desafio real era:

> gerar um arquivo que o Movavi realmente aceitasse.

## O que construímos

O gerador faz basicamente isto:

1. abre um template `.mepj`
2. extrai `config.json` e `meta.json`
3. encontra o clip de áudio principal e atualiza com o novo MP3
4. encontra os clips de imagem e troca os caminhos
5. recalcula o tempo com base na duração do áudio
6. sincroniza metadados reais das imagens, como largura, altura, formato e tamanho
7. preserva o estilo visual original sempre que possível
8. gera um novo `.mepj`

Na prática, isso me permite montar um novo projeto de short a partir de:

- 1 MP3
- 4 imagens
- 1 template base

## O ponto de virada

A grande virada foi perceber que **estrutura vale mais do que limpeza agressiva**.

No começo eu tentei remover sobras do template de forma muito agressiva: legendas herdadas, clips extras e itens antigos. Isso parecia lógico, mas o Movavi nem sempre gostava desse tipo de intervenção. Em alguns casos, remover demais direto no JSON fazia o projeto ficar instável.

O que funcionou melhor foi:

- preservar a estrutura do projeto
- substituir as mídias de forma conservadora
- manter uma base estável de template
- embutir um template de fallback dentro do próprio projeto

Esse fallback é importante porque evita depender de um arquivo externo frágil para o gerador continuar funcionando.

## O novo template padrão

Também criei um projeto-base novo:

- `padrao.mepj`

Esse arquivo virou o melhor template padrão porque já começa mais próximo do formato final que eu quero:

- 4 imagens
- sem legenda sobrando
- estrutura mais limpa

Ou seja: agora o gerador tem uma base mais confiável.

## Por que isso importa

Isso me ajuda porque eu produzo bastante conteúdo curto de boxe e esportes. Em vez de abrir o Movavi e refazer a mesma estrutura toda vez, agora eu posso:

- escolher a história
- gerar o áudio
- selecionar as imagens
- criar o projeto automaticamente
- revisar e exportar mais rápido

Isso economiza tempo e torna o fluxo repetível.

## Repositório

Se você quiser olhar o código, testar ou contribuir, o repositório é este:

<https://github.com/mauriciokj/shorts-project-generator>

## O que vem depois

Os próximos passos são:

- deixar o gerador ainda mais agnóstico ao template
- melhorar as regras de limpeza com segurança
- suportar mais variações de fluxo
- reduzir cada vez mais a edição manual dentro do Movavi

No fim das contas, a ideia não é substituir o Movavi. A ideia é fazer o Movavi funcionar como parte de um pipeline de produção repetível.
