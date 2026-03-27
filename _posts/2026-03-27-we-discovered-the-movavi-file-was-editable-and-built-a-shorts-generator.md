---
layout: default
title: "Descobrimos que o arquivo do Movavi era editável — e criamos um gerador de Shorts"
date: 2026-03-27 00:38:00 -0300
categories: [automation, video, productivity]
tags: [movavi, automation, python, shorts, video-editing, reverse-engineering]
---

Editar shorts em lote costuma parecer uma tarefa pequena — até você repetir o mesmo processo dezenas de vezes.

No nosso caso, o fluxo era sempre parecido:

- gerar o roteiro
- gerar o áudio
- separar imagens
- abrir o Movavi
- importar tudo
- ajustar a timeline
- revisar transições
- exportar

Nada disso era difícil isoladamente. O problema era a repetição.

A pergunta que começou a mudar tudo foi simples:

**e se o arquivo de projeto do Movavi fosse editável?**

Foi assim que começou um experimento que acabou virando algo bem mais interessante: uma prova de conceito funcional de um gerador de projetos para shorts.

---

## O problema: trabalho repetitivo demais para vídeos curtos

Quando você trabalha com vídeos curtos e formatos padronizados, percebe rápido que parte da edição não exige decisão criativa nova o tempo todo.

Muitas vezes, o processo é basicamente este:

- trocar o áudio principal
- trocar as imagens
- distribuir a duração na timeline
- manter um template visual já aprovado
- abrir no editor só para revisão final

Em outras palavras: muito do trabalho era operacional, não criativo.

A intuição era que isso deveria ser automatizável de alguma forma.

---

## A hipótese: o `.mepj` talvez não fosse uma caixa-preta

O Movavi salva seus projetos em arquivos com extensão `.mepj`.

A primeira pergunta era: esse arquivo é um blob proprietário impossível de mexer ou existe algo legível lá dentro?

A resposta foi melhor do que esperávamos.

Ao inspecionar o arquivo, descobrimos que o `.mepj` era, na prática, um **arquivo compactado** contendo pelo menos:

- `meta.json`
- `config.json`

Ou seja: o projeto não era uma caixa-preta total. Havia estrutura editável.

Esse foi o primeiro grande ponto de virada.

---

## O que encontramos dentro do projeto

O arquivo mais importante era o `config.json`.

Foi nele que apareceram os elementos mais úteis para automação:

- timeline
- clips
- caminhos absolutos dos arquivos usados
- duração dos elementos
- transições
- referências de áudio
- referências de imagem
- efeitos aplicados aos clipes
- até elementos de legenda herdados do template

Em outras palavras: o `config.json` funcionava como um mapa da edição.

Se o `.mepj` é a embalagem do projeto, o `config.json` é a planta da montagem.

---

## A primeira prova de conceito

Com isso em mãos, o primeiro teste foi simples:

1. pegar um projeto real já montado no Movavi
2. usar esse projeto como template
3. trocar programaticamente:
   - o arquivo de áudio
   - os caminhos das imagens
4. empacotar tudo de novo como `.mepj`
5. tentar abrir no Movavi

A intenção inicial nem era deixar o projeto perfeito.
Era só responder a pergunta:

**o Movavi vai aceitar um projeto gerado dessa forma?**

A resposta foi: **sim**.

O arquivo abriu.

Isso validou a ideia principal.

---

## O que ainda estava errado no começo

O primeiro projeto gerado abria, mas ainda estava longe do ideal.

Entre os problemas encontrados:

- nem todas as imagens eram substituídas corretamente
- a duração do áudio ainda herdava informação errada do template
- legendas do projeto original continuavam aparecendo
- parte da timeline ainda carregava restos do material anterior

Ou seja: a base era viável, mas ainda precisava entender melhor o comportamento interno do projeto.

---

## Descobrindo como o áudio e as imagens eram representados

Ao continuar a inspeção, ficou claro que o projeto armazenava coisas bem úteis para automação.

### No áudio
Foi possível encontrar:

- caminho do MP3
- tamanho do arquivo
- duração (`length`)
- duração do clip na timeline (`timing.duration`)
- duração de origem (`timing.sourceDuration`)

Isso permitiu uma melhoria importante:

em vez de reaproveitar a duração herdada do template, passamos a medir o MP3 real com `ffprobe` e sincronizar esses campos com o tempo verdadeiro do áudio.

### Nas imagens
Também foi possível localizar:

- paths dos PNGs/JPEGs
- clips visuais principais na trilha certa
- timestamp de entrada
- duração de cada imagem
- transições de entrada e saída

Com isso, conseguimos distribuir o tempo do vídeo com base no áudio.

Exemplo do raciocínio:

- se o áudio tem 55,4 segundos
- e o vídeo vai usar 4 imagens
- cada imagem recebe aproximadamente 1/4 da duração total

Isso transformou a montagem em algo previsível e automático.

---

## A legenda do template também dava para remover

No projeto-base havia um elemento de legenda herdado do template.

Esse clip aparecia com o nome:

```txt
#Subtitle_template#
```

Ao detectar isso no JSON, conseguimos simplesmente filtrar e remover esses clips antes de gerar o novo projeto.

Resultado: o arquivo gerado passou a abrir sem a legenda anterior contaminando a nova edição.

---

## E os efeitos? Também estavam lá

Outra descoberta importante foi que o template não guardava só caminhos de mídia.

Os clipes visuais também continham informações como:

- `effects`
- `cropEnvelope`
- `moveEnvelope`

Na prática, isso significava que o template já carregava parte da “linguagem visual” da edição:

- zoom
- enquadramento dinâmico
- efeitos aplicados sobre a imagem
- comportamento visual repetível

Ou seja: não era necessário reconstruir isso do zero. Em muitos casos, bastava preservar os envelopes e efeitos dos clips do template.

Isso é ótimo porque torna a automação muito mais útil. O projeto gerado não fica só “funcional”; ele já nasce com parte do acabamento visual.

---

## O resultado: um gerador de projetos de shorts

Com esses testes sucessivos, a prova de conceito deixou de ser apenas um hack isolado e virou uma ferramenta interna inicial.

Nasceu daí o:

## `shorts-project-generator`

A ideia da ferramenta é simples.

Você passa:

- um template `.mepj`
- um arquivo de áudio MP3
- uma lista de imagens
- um caminho de saída

E ela gera um novo `.mepj` com:

- áudio atualizado
- imagens atualizadas
- tempo distribuído entre as imagens
- legenda herdada removida
- transições básicas preservadas
- projeto pronto para abrir no Movavi

O objetivo não é substituir o editor.

O objetivo é eliminar o trabalho mecânico e deixar para o Movavi apenas a etapa final de revisão e exportação.

---

## O que já foi validado

Até aqui, o que conseguimos validar foi:

- o `.mepj` é editável
- o Movavi aceita projetos gerados a partir de template
- dá para trocar áudio e imagens programaticamente
- dá para medir a duração real do MP3 e ajustar a timeline
- dá para remover legenda herdada
- dá para preservar boa parte da estrutura visual do template
- dá para gerar um projeto utilizável na prática

E o melhor: **sem precisar de IA para isso**.

Essa parte é puramente automação.

---

## O que ainda pode evoluir

Mesmo com a PoC funcionando bem, ainda existem melhorias óbvias:

- mapear melhor todos os clips auxiliares do template
- tornar as regras de transição mais configuráveis
- preservar com mais inteligência os efeitos e zooms de cada imagem
- aceitar quantidades variáveis de imagens com mais flexibilidade
- criar uma interface simples em vez de usar apenas CLI

Mas o mais importante já aconteceu:

**a ideia deixou de ser hipótese e virou ferramenta funcional em beta.**

---

## Conclusão

O mais interessante desse experimento é que ele nasceu de uma pergunta operacional muito simples:

**“precisamos mesmo repetir isso manualmente toda vez?”**

A resposta foi não.

Ao investigar o formato do projeto do Movavi, encontramos uma brecha clara para automação. E, a partir de um template real, conseguimos gerar projetos novos com áudio, imagens, duração e estrutura básica já resolvidos.

Não é uma substituição do editor. Pelo menos não por enquanto.

Mas já é um atalho muito útil para um tipo de trabalho que, antes, consumia tempo demais para pouca decisão criativa real.

E esse tipo de automação costuma ser o melhor tipo: menos glamour, mais resultado.

---

## Quer testar o `shorts-project-generator`?

Se quiser testar o `shorts-project-generator`, comenta aí.
Se tiver interesse, eu coloco ele no GitHub para testes.
