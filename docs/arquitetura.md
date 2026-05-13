# Arquitetura do Lab TLOA

> Documentação técnica detalhada da infraestrutura do TLOA — configuração das máquinas virtuais, rede, componentes de segurança e decisões de design.

---

## Visão Geral da Infraestrutura

O lab do TLOA é construído sobre virtualização local usando VMware Workstation, com todas as máquinas operando em uma rede isolada (Host-Only). Essa escolha garante que o tráfego malicioso simulado nunca alcance redes externas.

---

## Rede

| Parâmetro | Valor |
|---|---|
| Tipo de rede | VMware Host-Only |
| Interface virtual | VMnet1 |
| Subnet | `192.168.204.0/24` |
| Gateway | Não configurado (isolado) |

A escolha por Host-Only é intencional: simula um ambiente corporativo fechado e impede vazamento de tráfego para a rede doméstica ou internet.

---

## Máquinas Virtuais

### DC01 — Domain Controller

| Parâmetro | Valor |
|---|---|
| Sistema operacional | Windows Server 2025 |
| Hostname | `DC01` |
| IP | `192.168.204.132` |
| Domínio | `tloa.local` |
| Serviços | AD DS, DNS, Sysmon, Wazuh Agent 4.7.5 |
| Wazuh Agent ID | 002 |

**Funções no lab:**
- Controlador de domínio principal (`tloa.local`)
- Servidor DNS para resolução interna
- Alvo principal dos ataques simulados (Kerberoasting, enumeração AD, credential dumping)
- Fonte de telemetria via Sysmon + Wazuh Agent

**Notas de configuração:**
- Windows Server 2025 depreca RC4 por padrão — afeta técnicas de Kerberoasting que dependem de tickets RC4
- LDAP signing enforçado por padrão — bloqueia consultas LDAP não assinadas (impacta ferramentas de enumeração)
- Ambas as restrições foram documentadas como findings técnicos realistas nos exercícios

---

### WS01 — Workstation

| Parâmetro | Valor |
|---|---|
| Sistema operacional | Windows 10 |
| Hostname | `WS01` |
| Domínio | `tloa.local` (joined) |
| Serviços | Sysmon, Wazuh Agent |

**Funções no lab:**
- Máquina vítima para simulações de comprometimento inicial
- Fonte de telemetria de endpoint
- Ponto de lateral movement nos cenários de IR

---

### SIEM — Wazuh Manager

| Parâmetro | Valor |
|---|---|
| Sistema operacional | Linux (Ubuntu) |
| Versão | Wazuh 4.7.5 |
| Função | SIEM & XDR |

**Fontes de log configuradas:**
- Windows Event Logs (Security, System, Application)
- Sysmon (via canal Microsoft-Windows-Sysmon/Operational)
- Syslog (agentes Linux, se adicionados futuramente)

**Dashboards:**
- ATT&CK-mapped alert views
- Agent health monitoring
- Custom rules para técnicas do Atomic Red Team

---

## Componentes de Segurança

### Sysmon

- **Configuração:** SwiftOnSecurity (config pública, amplamente utilizada em labs e produção)
- **Instalado em:** DC01, WS01
- **Eventos cobertos:** Criação de processos, conexões de rede, modificações de registro, criação de arquivos, pipe events

A config do SwiftOnSecurity oferece cobertura ampla com baixo ruído — adequada para detectar as técnicas do Atomic Red Team sem gerar volume excessivo de eventos.

### Wazuh Agent 4.7.5

- **Instalado em:** DC01 (Agent ID: 002), WS01
- **Coleta:** Sysmon + Windows Event Logs
- **Encaminhamento:** Wazuh Manager via protocolo ossec-remoted (porta 1514 UDP/TCP)

### BloodHound (Community Edition)

- **Uso:** Enumeração de attack paths no Active Directory
- **Ingestão:** SharpHound collector executado em WS01
- **Dados coletados:** ACLs, memberships, SPNs, sessões ativas, Kerberoastable accounts

### Atomic Red Team

- **Uso:** Execução de técnicas ATT&CK de forma padronizada e reproduzível
- **Execução:** PowerShell (via Invoke-AtomicTest)
- **Mapeamento:** Cada teste referencia um ID ATT&CK diretamente

---

## Active Directory — Design Intencional

O ambiente AD foi configurado com vulnerabilidades intencionais para simular superfícies de ataque reais encontradas em ambientes corporativos:

### Contas e Misconfigurations

| Conta | Tipo | Misconfiguration | Técnica relacionada |
|---|---|---|---|
| `jsmith` | Usuário padrão | Senha fraca, reutilização de credenciais | T1078, T1110 |
| `sconnor` | Usuário padrão | Membro de grupos sensíveis | T1069, T1087 |
| `svc-backup` | Service account | SPN configurado (`MSSQLSvc/dc01.tloa.local:1433`) | T1558.003 (Kerberoasting) |

### Limitações Conhecidas (Windows Server 2025)

O Windows Server 2025 introduz hardening por padrão que impacta técnicas clássicas:

| Restrição | Impacto | Documentação |
|---|---|---|
| RC4 deprecation | Kerberoasting bloqueado para tickets RC4 | Documentado em TLOA-Lab-Exercises |
| LDAP signing enforçado | Ferramentas de enumeração LDAP falham sem assinatura | Documentado em TLOA-Lab-Exercises |

Essas restrições não foram contornadas — foram documentadas como findings técnicos realistas, refletindo o comportamento de ambientes modernos.

---

## Pipeline de Telemetria

```
DC01 / WS01
    │
    ├── Sysmon (SwiftOnSecurity config)
    │       └── Eventos: Process Create, Network, Registry, File, Pipe
    │
    └── Wazuh Agent 4.7.5
            └── Coleta: Sysmon + Windows Event Logs
                    │
                    ▼
            Wazuh Manager (SIEM)
                    │
                    ├── Alertas ATT&CK-mapeados
                    ├── Custom rules (Atomic Red Team)
                    └── Dashboards de detecção
```

---

## Decisões de Design

**Por que Windows Server 2025?**
Ambientes modernos utilizam versões recentes. Usar WS2025 força a documentação de comportamentos de hardening reais, tornando o lab mais fiel à realidade operacional do que usar versões legadas permissivas.

**Por que Host-Only networking?**
Isolamento total. Tráfego de C2, Kerberoasting, lateral movement e credential dumping não devem alcançar redes externas. Além disso, reflete o modelo de rede interna corporativa que o lab simula.

**Por que Wazuh?**
Open-source, amplamente usado em ambientes reais no Brasil e internacionalmente, com suporte nativo a regras ATT&CK e integração com Sysmon. Alinhado ao perfil de vagas SOC/Blue Team que o lab suporta.

**Por que Atomic Red Team?**
Reproduzibilidade. Cada técnica é executada com parâmetros documentados e referência ATT&CK direta, permitindo que qualquer pessoa reproduza o cenário e valide a detecção.

---

*Documentação mantida em português. README principal disponível em inglês no repositório raiz.*
