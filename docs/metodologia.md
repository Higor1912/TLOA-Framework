# Metodologia TLOA

> Documentação detalhada das 7 fases do framework Threat-Led Offensive Audit — da autorização ao relatório final.

---

## Visão Geral

O TLOA é estruturado como um ciclo sequencial de 7 fases. Cada fase tem um objetivo claro, entradas definidas e outputs concretos. O ciclo pode ser executado de ponta a ponta para um exercício completo, ou fases individuais podem ser repetidas conforme novas ameaças são identificadas ou novos cenários são modelados.

```
Fase 0 — Scoping
    ↓
Fase 1 — Contexto
    ↓
Fase 2 — Inteligência de Ameaças
    ↓
Fase 3 — Modelagem de Cenários
    ↓
Fase 4 — Validação Técnica
    ↓
Fase 5 — Priorização de Riscos
    ↓
Fase 6 — Relatório
```

---

## Fase 0 — Scoping

**Objetivo:** Definir os limites do exercício antes de qualquer execução técnica.

Todo exercício começa com autorização explícita e escopo documentado. Mesmo em ambiente de home lab pessoal, essa fase é praticada como um hábito operacional — refletindo o processo real de um engagement profissional.

**Atividades:**
- Definir o ambiente alvo (quais VMs, quais IPs, qual domínio)
- Estabelecer as regras de engajamento (o que pode ser executado, o que está fora do escopo)
- Documentar o período de execução
- Identificar os objetivos do exercício (o que se quer validar)

**Output:** `rules-of-engagement.md` — documento de autorização e escopo do exercício.

**No contexto do TLOA lab:**
O ambiente alvo é sempre a rede `192.168.204.0/24` (VMnet1 Host-Only). Técnicas destrutivas (wiper, ransomware simulado) são fora do escopo por padrão. O DC01 pode ser alvo de enumeração e exploração, mas não de ataques que comprometam a disponibilidade da VM.

---

## Fase 1 — Contexto

**Objetivo:** Mapear o ambiente alvo para entender a superfície de ataque antes de selecionar cenários.

Antes de simular um ataque, é necessário entender o que existe para ser atacado. Essa fase mapeia ativos, serviços, contas e configurações — da mesma forma que um atacante real faria em reconhecimento inicial.

**Atividades:**
- Inventário de ativos (VMs, IPs, hostnames, sistemas operacionais)
- Mapeamento de serviços expostos (portas, protocolos, versões)
- Identificação de contas de domínio e seus privilégios
- Levantamento de misconfigurations conhecidas (SPNs Kerberoastáveis, senhas fracas, grupos sensíveis)
- Mapeamento da topologia de rede

**Output:** `asset-inventory.md` — inventário documentado do ambiente com superfície de ataque mapeada.

**Ferramentas usadas no TLOA lab:**
- `nmap` — mapeamento de serviços e portas
- `BloodHound + SharpHound` — enumeração do Active Directory (attack paths, ACLs, SPNs)
- `net user`, `net group` — enumeração manual de contas e grupos

---

## Fase 2 — Inteligência de Ameaças

**Objetivo:** Identificar threat actors relevantes e extrair as TTPs que serão usadas na simulação.

Essa é a fase que diferencia o TLOA de um simples pentest genérico. Em vez de executar técnicas aleatórias, o framework parte de threat intelligence real: quais grupos atacam organizações com o perfil do ambiente simulado? Quais TTPs eles usam? Quais já foram documentadas com evidências públicas?

**Atividades:**
- Identificar threat actors relevantes para o setor/perfil do ambiente simulado
- Levantar TTPs documentadas (MITRE ATT&CK, relatórios de threat intel, ISACs)
- Selecionar as técnicas que serão priorizadas no exercício
- Documentar as fontes de inteligência utilizadas

**Output:** `threat-profile.md` — perfil do threat actor selecionado com TTPs mapeadas e fontes.

**Fontes utilizadas no TLOA:**
- MITRE ATT&CK (attack.mitre.org)
- Relatórios públicos de threat intel (Mandiant, CrowdStrike, Microsoft MSTIC)
- CTI Toolkit — módulos de enriquecimento e mapeamento de TTPs via STIX 2.1
- Repositório `Threat-Intel-Reports` — reports semanais com tracking de threat actors ativos

**Exemplo aplicado:**
No exercício TLOA-IR-Cases Case 004 (Pass-the-Hash), o threat actor de referência foi um grupo de ransomware que usa T1550.002 para lateral movement após comprometimento inicial. A escolha de WinRM (T1021.006) como vetor de movimento lateral foi baseada em TTPs documentadas desse perfil.

---

## Fase 3 — Modelagem de Cenários

**Objetivo:** Transformar as TTPs identificadas em cenários de ataque concretos e executáveis.

A modelagem conecta a inteligência de ameaças à execução técnica. Cada cenário define o caminho de ataque completo — do ponto de entrada ao objetivo final — antes de qualquer comando ser executado.

**Atividades:**
- Definir o objetivo do cenário (o que o atacante está tentando alcançar)
- Selecionar as TTPs que compõem o attack path
- Mapear as ferramentas que serão usadas para cada técnica
- Identificar quais artefatos serão gerados (logs, arquivos, eventos)
- Antecipar quais detecções deveriam disparar (hipóteses de detecção)

**Output:** `attack-scenario.md` — documento com o caminho de ataque completo, TTPs selecionadas, ferramentas e hipóteses de detecção.

**Estrutura de um cenário TLOA:**

```
Cenário: [Nome descritivo]
Threat actor referência: [Ex: FIN7, Kimsuky, grupo genérico de ransomware]
Objetivo: [Ex: Lateral movement de WS01 para DC01 via credenciais comprometidas]

Attack path:
  1. [Técnica] → [Ferramenta] → [Artefato esperado]
  2. [Técnica] → [Ferramenta] → [Artefato esperado]
  ...

Hipóteses de detecção:
  - [O Wazuh deveria alertar X via regra Y]
  - [Sysmon EID Z deveria ser gerado]
```

---

## Fase 4 — Validação Técnica

**Objetivo:** Executar os cenários modelados e documentar evidências — tanto de detecção quanto de gaps.

Essa é a fase de execução. Cada técnica é executada conforme o cenário modelado, e os resultados são documentados sistematicamente: o que foi executado, qual foi o resultado, o que o SIEM detectou, e o que passou despercebido.

**Atividades:**
- Executar as técnicas seguindo o attack path modelado
- Capturar evidências de execução (screenshots, logs, output de comandos)
- Analisar alertas gerados no Wazuh em tempo real
- Correlacionar eventos no Sysmon com as regras disparadas
- Documentar gaps — técnicas executadas com sucesso sem geração de alerta

**Output:** Evidências técnicas organizadas + `detection-gap-analysis.md` com mapeamento completo de o que foi detectado e o que não foi.

**Fluxo de execução no TLOA lab:**

```
Selecionar técnica (ATT&CK ID)
    ↓
Executar via Atomic Red Team ou manual
    ↓
Monitorar Wazuh em tempo real
    ↓
Se alerta gerado → documentar regra, severidade, mapeamento ATT&CK
Se sem alerta → documentar como gap + analisar por quê (sem regra? cobertura de log insuficiente? evasão?)
    ↓
Capturar evidências (Wazuh dashboard, terminal output)
    ↓
Registrar no relatório de exercício
```

**Ferramentas de execução no TLOA lab:**
- `Invoke-AtomicTest` (Atomic Red Team) — execução padronizada de TTPs
- `Mimikatz` — credential dumping (T1003.001, T1550.002)
- `Netexec` / `Evil-WinRM` — lateral movement (T1021.006)
- `BloodHound` — enumeração de attack paths pós-comprometimento

---

## Fase 5 — Priorização de Riscos

**Objetivo:** Classificar os gaps e findings por risco real, não apenas por severidade técnica isolada.

Após a validação, nem todo gap tem a mesma urgência. Essa fase analisa cada finding à luz do threat actor de referência, da probabilidade de exploração e do impacto potencial para o ambiente.

**Critérios de priorização:**

| Critério | Peso |
|---|---|
| O threat actor de referência usa essa técnica ativamente? | Alto |
| A técnica foi executada com sucesso no lab? | Alto |
| Qual o impacto potencial (acesso a credenciais, lateral movement, persistência)? | Alto |
| Existe uma mitigação simples disponível? | Médio |
| A técnica requer acesso privilegiado prévio? | Médio |

**Output:** Risk ranking documentado — lista priorizada de gaps com justificativa baseada em threat context.

**Exemplo de priorização aplicada no TLOA:**

| Finding | Técnica | Prioridade | Justificativa |
|---|---|---|---|
| LSASS dump sem detecção | T1003.001 | 🔴 Crítica | Usado por APTs e ransomware; acesso a credenciais em texto claro; sem regra nativa no Wazuh |
| Recon de rede sem visibilidade | T1046 | 🟡 Média | Firewall bloqueou execução; gap está na camada de rede, não no endpoint |
| Latência do syscheck | T1112 | 🟡 Média | Detectado, mas com 12h de atraso; mitigação simples via Sysmon EID 13 |

---

## Fase 6 — Relatório

**Objetivo:** Comunicar os resultados com clareza para dois públicos distintos: executivo e técnico.

O relatório é o produto final de cada ciclo TLOA. Ele não documenta apenas o que foi executado — documenta o risco real que os gaps representam e as ações concretas para mitigá-los.

**Estrutura do relatório TLOA:**

```
1. Sumário Executivo
   └── O que foi testado, resultado geral, principais riscos identificados

2. Escopo e Metodologia
   └── Ambiente, período, fases executadas, threat actor de referência

3. Resultados por Técnica
   └── Tabela ATT&CK: técnica | executada | detectada | observação

4. Gaps de Detecção
   └── Descrição técnica + impacto potencial + recomendação de mitigação

5. Risk Ranking
   └── Findings priorizados por risco real

6. Conclusão e Próximos Passos
   └── O que será corrigido, o que será retestado, próximas TTPs a validar
```

**Formatos de output no TLOA:**
- `ir-case-XXX.md` — relatório técnico completo (repositório `TLOA-IR-Cases`)
- PDF exportado via `ti_report_builder_v2.html` (CTI Toolkit)
- Sumário executivo em formato Markdown para publicação

---

## O Ciclo de Reteste

O TLOA não termina no relatório. Após mitigações implementadas, o ciclo recomeça a partir da Fase 4 — reexecutando as mesmas técnicas para validar se os gaps foram corrigidos. Esse loop de validação → correção → revalidação é o que transforma exercícios isolados em maturidade operacional contínua.

```
Fase 4 (Validação) → Gap identificado
    ↓
Implementar mitigação (regra customizada, config, controle)
    ↓
Fase 4 (Reteste) → Confirmar detecção
    ↓
Fase 6 (Relatório atualizado) → Documentar fechamento do gap
```

---

## Conexão com os Outros Repositórios

```
TLOA-Framework (este repositório)
    └── Define a metodologia e a arquitetura

TLOA-Lab-Exercises
    └── Executa as Fases 3 e 4 — cenários modelados e validação técnica

TLOA-IR-Cases
    └── Executa as Fases 4, 5 e 6 — investigação, priorização e relatório

CTI-Toolkit
    └── Suporta a Fase 2 — mapeamento de TTPs e enriquecimento de threat intel

Threat-Intel-Reports
    └── Alimenta a Fase 2 — tracking contínuo de threat actors ativos
```

---

*Documentação mantida em português. README principal disponível em inglês no repositório raiz.*
