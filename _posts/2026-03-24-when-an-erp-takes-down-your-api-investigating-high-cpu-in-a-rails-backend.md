---
layout: default
title: "Quando um ERP derruba sua API: investigando CPU alta em um backend Rails"
date: 2026-03-24 22:28:00 -0300
categories: [rails, performance, backend]
tags: [rails, ruby, puma, sidekiq, performance, observability, erp, api]
---

Recentemente enfrentei um problema clássico — e ao mesmo tempo muito chato de diagnosticar — em um backend Rails:

- CPU constantemente em 100%
- API lenta
- workers saturados
- load average muito acima do normal
- vendedores reclamando de lentidão no app

O sistema em questão era uma API Rails consumida por três frentes principais:

- aplicativo mobile dos vendedores
- integração com ERP
- jobs assíncronos com Sidekiq

Neste post vou mostrar, de forma prática, como investigamos o problema, quais hipóteses levantamos, o que medimos, como encontramos a causa raiz e quais decisões de arquitetura foram tomadas.

---

## O cenário inicial

O primeiro alerta veio do monitoramento do servidor.

Ao abrir o `htop`, o quadro era bem ruim:

- CPU constantemente entre 90% e 100%
- load average acima de 10 em uma máquina com 4 CPUs
- muitos processos Ruby ativos
- workers do Puma consumindo CPU sem parar

Um retrato típico era algo assim:

```txt
CPU 0: 95%
CPU 1: 94%
CPU 2: 92%
CPU 3: 97%

Load average: 11.65 10.41 10.04
```

Isso já deixava claro que havia saturação real da máquina. A dúvida era: **o problema era concorrência, job, banco, app mobile ou alguma integração externa?**

---

## Primeira hipótese: Sidekiq saturando CPU

Ao olhar os processos ativos, vários deles eram:

- `sidekiq`
- `ruby`
- `puma worker`

A primeira suspeita foi a mais natural:

- jobs pesados demais
- concurrency alta demais
- fila congestionada

A configuração do Sidekiq foi reduzida.

### Antes

```txt
16 threads
```

### Depois

```txt
8 threads
```

### Resultado

- houve uma melhora pequena
- a CPU continuou saturada

### Conclusão

**O Sidekiq contribuía para a pressão da máquina, mas claramente não era a causa principal.**

---

## Segunda hipótese: Puma agressivo demais

A configuração encontrada no Puma também chamou atenção:

- **5 workers**
- **20 threads**

Concorrência total teórica:

```txt
5 x 20 = 100 threads
```

Para uma máquina com 4 CPUs, isso é agressivo demais.

Reduzimos:

- quantidade de workers
- quantidade de threads

### Resultado

- nova melhora pequena
- CPU ainda alta

### Conclusão

**Não era apenas um problema de concorrência excessiva no Puma.**

---

## Investigando requisições lentas

O próximo passo foi olhar o `production.log`.

Encontramos linhas como estas:

```txt
Completed 200 OK in 3004ms
Completed 200 OK in 1492ms
Completed 200 OK in 1439ms
Completed 200 OK in 3603ms
```

Ou seja, havia requests levando:

- 1,4s
- 3s
- 3,6s

O detalhe mais importante aparecia no breakdown do Rails:

```txt
ActiveRecord: 2510ms
```

Isso sugeria:

- queries pesadas
- banco sendo muito pressionado
- aplicação passando boa parte do tempo em acesso a dados

Mas ainda faltava responder o principal:

**quem estava gerando tantas requisições?**

---

## Investigando os endpoints mais usados

Filtramos os logs por controller:

```bash
grep "Processing by" production.log
```

Os endpoints que mais apareciam incluíam:

- `ProductsController`
- `OrdersController`
- `SessionsController#create`

E aí veio o ponto que realmente chamou atenção:

```txt
DeviseTokenAuth::SessionsController#create
```

Ou seja: **login**.

---

## O número que entregou o problema

Rodamos a contagem de logins:

```bash
grep "SessionsController#create" production.log | wc -l
```

Resultado:

```txt
464529
```

Mais de **460 mil logins**.

Isso era absurdo.

Para comparar, um endpoint comum como listagem de produtos tinha algo como:

```txt
ProductsController#index → 31.533
```

Ou seja:

- havia muito mais login do que uso real do sistema
- alguma integração estava autenticando em loop
- o sistema estava gastando CPU demais com uma operação cara e repetitiva

---

## Separando mobile e ERP

A API tinha duas versões principais:

- `/api/v1` → ERP
- `/api/v6` → app mobile

Separando os logins por versão, o padrão ficou escancarado.

### App mobile

```txt
/api/v6/auth/sign_in → 2.049
```

### ERP

```txt
/api/v1/auth/sign_in → 465.833
```

### Conclusão

**O problema não era o app mobile. O problema era a integração com o ERP.**

---

## O padrão suspeito

Ao continuar olhando o log, o comportamento era repetitivo:

```txt
POST /api/v1/auth/sign_in
POST /api/v1/auth/sign_in
POST /api/v1/auth/sign_in
POST /api/v1/auth/sign_in
```

Chamadas múltiplas por segundo.

Isso indicava um padrão clássico:

- loop de login
- token não sendo reutilizado
- autenticação sendo feita de forma excessiva

E login em Rails não é barato:

- busca usuário
- valida credenciais
- gera token
- faz hash
- serializa resposta

Esse tipo de operação repetida em volume alto explica muito bem CPU estourando.

---

## Confirmando com `perf`

Para ter mais confiança, rodamos `perf top`.

O resultado trouxe funções como:

- `vm_sendish`
- `vm_exec_core`
- `rb_mutex_trylock`
- `pthread_mutex_unlock`

Isso apontava para:

- CPU sendo consumida no interpretador Ruby
- contenção entre threads
- alta concorrência em operação Ruby-heavy

### Conclusão técnica

- Ruby realmente estava saturando CPU
- havia muitas requisições simultâneas
- o padrão de login massivo confirmava a hipótese principal

---

## A arquitetura agravava o problema

Naquele momento, tudo rodava no mesmo servidor:

- Rails
- Puma
- Sidekiq
- PostgreSQL
- tráfego do ERP
- tráfego do app mobile

Isso gerava competição por:

- CPU
- memória
- conexões
- tempo de resposta

Na prática, quando o ERP sobrecarregava a API:

**o app mobile também sofria.**

Esse é o tipo de arquitetura que funciona enquanto a carga está controlada. Quando uma integração externa se comporta mal, tudo vai junto para o chão.

---

## A decisão tomada: isolar a carga

A decisão foi separar responsabilidades em servidores diferentes.

### Arquitetura adotada

**Servidor 1**
- ERP
- banco

**Servidor 2**
- API mobile

Ambos acessando:

- banco compartilhado

### Benefícios imediatos

- isolamento de carga
- proteção do app mobile
- CPU dedicada para cada papel
- base melhor para escalar depois

---

## E a latência de rede?

Essa foi uma preocupação natural.

Sim, ao separar servidores, existe custo de rede. Algo como:

- +20ms
- +40ms

Mas isso é pequeno perto do que já estava acontecendo com a máquina saturada:

- +500ms
- +1000ms
- +3000ms em requests lentos

### Em outras palavras

**Mesmo adicionando latência de rede, o sistema ainda melhora quando você remove o gargalo de CPU e o contágio entre cargas.**

---

## Antes e depois

### Antes

```txt
App + ERP + DB
```

### Depois

```txt
App Server
   |
   |
ERP + DB
```

Ainda não é a arquitetura final ideal, mas já representa um ganho importante de estabilidade e previsibilidade.

---

## Solução temporária e solução definitiva

### Solução temporária

- separar servidores
- isolar a carga do ERP
- proteger a API mobile

### Solução definitiva

- corrigir a integração do ERP
- reutilizar token corretamente
- evitar login repetido
- reduzir chamadas desnecessárias

### Arquitetura futura ideal

```txt
App Server
ERP Server
Sidekiq Server
Database Server
```

Esse desenho reduz acoplamento operacional e ajuda a escalar cada peça de forma independente.

---

## Lições aprendidas

## 1. CPU alta nem sempre é problema de hardware

Quando a CPU está em 100%, a reação instintiva costuma ser pensar em upgrade de máquina.

Mas muitas vezes o problema está em:

- comportamento da aplicação
- padrão de integração
- concorrência mal calibrada
- operação repetitiva demais

Infraestrutura maior não resolve arquitetura ruim. No máximo, adia o problema.

---

## 2. Logs continuam sendo uma das melhores ferramentas de diagnóstico

Com ferramentas simples como:

- `grep`
- `wc`
- `perf`
- `htop`

foi possível descobrir:

- a origem da carga
- o tipo de operação mais custosa
- quem estava disparando o problema
- qual decisão de arquitetura fazia sentido

Antes de sair trocando componente, vale sempre extrair o máximo de evidência do que já está disponível.

---

## 3. Integrações externas podem derrubar o seu sistema inteiro

Mesmo quando:

- o app está saudável
- o banco está saudável
- o deploy está correto

uma integração mal implementada pode:

- saturar CPU
- gerar milhares de requests inúteis
- degradar a experiência de outros consumidores da API

Esse tipo de incidente é menos sobre bug isolado e mais sobre **falta de isolamento de carga**.

---

## Conclusão

Depois da investigação, o quadro ficou claro:

- CPU alta confirmada
- login massivo detectado
- ERP identificado como origem
- Ruby saturando CPU confirmado via `perf`
- arquitetura isolada adotada como mitigação imediata

Separar servidores foi a decisão correta naquele momento porque entregou:

- melhora imediata
- mais estabilidade
- proteção ao app mobile
- caminho de escalabilidade mais claro

Este caso é um ótimo exemplo de um problema que parece ser “infra”, mas na verdade nasce da combinação entre:

- observabilidade
- comportamento de integração
- desenho de arquitetura

No fim, foi um caso clássico de:

**problema de arquitetura resolvido com observabilidade, isolamento de carga e leitura fria dos logs.**
