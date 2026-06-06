---
theme: seriph
transition: slide-up
layout: intro
comark: true
---

<SlideNum />

<div class="flex items-center gap-4">
  <h1>Simulação de Monte Carlo</h1>
  <img src="/assets/catolica.png" class="w-32 h-32">
</div>

### Matéria: Pesquisa Operacional

### Professor: Jaisson Potrich dos Reis

#### Alunos:

- Henrique Maia Cardosa
- João Miguel de Castro Menna
- João Pedro Medeiros Izidoro
- Ricardo Gabriel Fialho Santos

---
transition: slide-up
---

<SlideNum />

# O que é Simulação de Monte Carlo?

- Técnica de modelagem e análise 
- Utiliza a geração de números aleatórios para simular o comportamento de sistemas complexos.
- Ela é amplamente utilizada em diversas áreas para avaliar riscos, tomar decisões e prever resultados futuros.

<br />
<br />

# Nome "Monte Carlo"

- O nome "Monte Carlo" é uma referência à cidade de Monte Carlo, conhecida por seus cassinos e jogos de azar, onde a aleatoriedade desempenha um papel fundamental.

---
transition: slide-up
---

<SlideNum />

# Problemas a serem resolvidos

<div class="flex gap-4">
  <ProblemCard title="Repovoamento de Aves" class="mb-4">
    <span>Equilibrar a disponibilidade de aves para repovoamento dos parques nacionais.</span>
    <template #image>
      <img src="/assets/balanca-rabo-leitoso.jpg" class="w-full h-64 object-cover rounded-md">
    </template>
  </ProblemCard>

  <ProblemCard title="Gestão de Riscos" class="mb-4">
    <span>Planejar a gestão de riscos em um software de pagamentos.</span>
    <template #image>
      <img src="/assets/gestao-riscos.webp" class="w-full h-64 object-cover rounded-md">
    </template>
  </ProblemCard>
</div>

---
transition: slide-up
layout: center
---

<SlideNum />

# Case 1: Repovoamento de Balança-Rabo-Leitoso's

---
transition: slide-up
layout: default
---

<SlideNum />

# Centro de Conservação de Balança-Rabo-Leitoso

- Uma ave ameaçada de extinção (nesse universo).
- Desafio: manter um estoque adequado de aves para repovoamento dos parques nacionais.
- A demanda por aves é incerta e os custos de estoque, falta e pedidos são significativos.

<br />

<div class="opacity-85 mb-2">
  Controle de estoque (s, Q) (sistema de revisão contínua) testa comportamento do estoque sob condições de incerteza.
</div>

<div class="opacity-85 mb-2">
</div>

<div class="opacity-85 mb-2">
  Objetivo: testar a política <strong>(s, Q)</strong> para equilibrar custo e nível de serviço em 52 semanas.
</div>

---
transition: slide-up
---

<SlideNum />

<div class="flex gap-4 text-sm mb-3 h-full">
  <div class="flex flex-col gap-4">
    <Card title="Política (s, Q)">
      <span>Variáveis de decisão:</span>
      <ul>
        <li>s (ponto de pedido) = 4</li>
        <li>Q (quantidade a ser reposta) = 6</li>
      </ul>
    </Card>
    <Card title="Desafios">
      <ul>
        <li>Demanda semanal incerta ~ Poisson (&lambda; = 3)</li>
        <li>Lead time variável ~ 1 a 4 semanas com maior frequência em 2 semanas</li>
        <li>Custos de estoque (R$200), falta (R$1000) e pedidos (R$500)</li>
      </ul>
    </Card>
    <Card title="Saídas da simulação">
      <ul>
        <li>Custo médio, mediana e risco P95</li>
        <li>Nível de serviço (% semanas sem falta)</li>
        <li>Distribuição de custos anuais</li>
      </ul>
    </Card>
  </div>
  <Card class="flex flex-col gap-4 items-center justify-center">
    <span class="text-lg font-bold text-gray-800">Simulação do Centro de Conservação</span>
    <img src="/assets/balanca-rabo-leitoso-2.jpg" class="w-96 h-96 object-cover rounded-md">
  </Card>
</div>

---
title: "Simulação"
transition: slide-up
---

<AnimalsMonteCarloLive />

---
transition: slide-up
layout: center
---

<SlideNum />

# Case 2: Gestão de Riscos em Software de Pagamentos

---
transition: slide-up
---

<SlideNum />

# Corrida contra o tempo

- O prazo fixo para entrega do software é de 90 dias.
- Multas pesadas em caso de atraso.
- Dividido em 3 fases: integração, desenvolvimento e implantação.

---
transition: slide-up
---

<SlideNum />

# Fases do Projeto

<div class="flex flex-col gap-4 w-full">
  <Card title="Fase 1: Integração">
    <span>Atividades: integração de sistemas, configuração inicial, testes preliminares.</span>
    <div class="flex gap-4">
      <span>Média: <strong>30 dias</strong></span>
      <span>Máximo: <strong>45 dias</strong></span>
    </div>
  </Card>
  <Card title="Fase 2: Desenvolvimento">
    <span>Atividades: codificação, revisão de código, integração contínua.</span>
    <div class="flex gap-4">
      <span>Média: <strong>35 dias</strong></span>
      <span>Máximo: <strong>55 dias</strong> (alta incerteza)</span>
    </div>
  </Card>
  <Card title="Fase 3: Implantação">
    <span>Atividades: configuração de servidores, implantação em produção, monitoramento inicial.</span>
    <div class="flex gap-4">
      <span>Média: <strong>15 dias</strong></span>
      <span>Máximo: <strong>30 dias</strong></span>
    </div>
  </Card>
</div>

---
transition: slide-up
---

<SlideNum />

# Riscos

<div class="flex flex-col gap-4 w-full">
  <Card title="Risco 1: Auditoria (30% de chance)">
    <span>Auditoria externa pode atrasar o projeto.</span>
    <span>Impacto: se ocorrer, atrasa entre 10 e 20 dias adicionais.</span>
  </Card>
  <Card title="Risco 2: Falha da API (15% de chance)">
    <span>Falha da API pode interromper o desenvolvimento.</span>
    <span>Impacto: se ocorrer, atrasa entre 12 dias adicionais.</span>
  </Card>
</div>

---
title: "Simulação do Case 2"
transition: slide-up
---

<FinTechMonteCarloLive />

---
transition: slide-up
layout: center
---

<SlideNum />

# Obrigado!

## Estamos abertos a perguntas.
