---
theme: seriph
transition: slide-up
layout: intro
comark: true
---

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

# Problemas a serem resolvidos

<div class="flex gap-4">
  <ProblemCard title="Repovoamento de Aves" class="mb-4">
    <span>Equilibrar a disponibilidade de aves para repovoamento dos parques nacionais.</span>
  </ProblemCard>

  <ProblemCard title="Gestão de Riscos" class="mb-4">
    <span>Planejar a gestão de riscos em um software de pagamentos.</span>
  </ProblemCard>
</div>

---
transition: slide-up
layout: default
---

# Centro de Conservacao de Fenix-do-Cerrado

<div class="text-sm opacity-85 mb-2">
  Objetivo: testar a politica <strong>(s, Q)</strong> para equilibrar custo e nivel de servico em 52 semanas.
</div>

<div class="grid grid-cols-2 gap-4 text-sm mb-3">
  <div class="rounded-lg border border-zinc-300 bg-zinc-50 p-3">
    <h3 class="font-bold mb-1">Incertezas modeladas</h3>
    <ul>
      <li>Demanda semanal ~ Poisson (&lambda; = 3)</li>
      <li>Lead time ~ Triangular (1, 2, 4) semanas</li>
      <li>Custos: estoque R$200, falta R$1000, pedido R$500</li>
    </ul>
  </div>
  <div class="rounded-lg border border-zinc-300 bg-zinc-50 p-3">
    <h3 class="font-bold mb-1">Saidas da simulacao</h3>
    <ul>
      <li>Custo medio, mediana e risco P95</li>
      <li>Nivel de servico (% semanas sem falta)</li>
      <li>Distribuicao de custos anuais</li>
    </ul>
  </div>
</div>

---
title: "Simulação"
transition: slide-up
---

<MonteCarloLive />