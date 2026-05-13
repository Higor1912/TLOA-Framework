# Cobertura ATT&CK — TLOA Lab

> Tabela consolidada de todas as técnicas executadas ou tentadas ao longo dos exercícios e IR Cases do TLOA, com status de detecção, gaps identificados e referência ao repositório de origem.

---

## Legenda

| Símbolo | Significado |
|---|---|
| ✅ | Detectado em tempo real pelo Wazuh |
| ⚠️ | Detectado com latência ou após intervenção manual |
| ❌ | Não detectado — gap confirmado |
| 🚫 | Bloqueado pelo ambiente antes da execução (hardening) |

---

## Tabela de Cobertura

| ATT&CK ID | Técnica | Tática | Executada | Detectada | Ferramenta | Observação | Repositório |
|---|---|---|---|---|---|---|---|
| T1046 | Network Service Discovery | Reconnaissance | ✅ | ❌ | nmap | Firewall do Windows bloqueou o scan — gap na camada de rede; Wazuh sem visibilidade sobre tentativas bloqueadas | TLOA-Lab-Exercises |
| T1053.005 | Scheduled Task/Job | Persistence, Privilege Escalation | ✅ | ✅ | Atomic Red Team | Wazuh Rule 61104 disparada em tempo real; mapeamento automático para Persistence + Privilege Escalation | TLOA-Lab-Exercises |
| T1112 | Modify Registry | Defense Evasion | ✅ | ⚠️ | Atomic Red Team | Detectado via syscheck com latência de 12h; detecção em tempo real requer Sysmon EID 12/13 | TLOA-Lab-Exercises |
| T1548.002 | Abuse Elevation Control Mechanism — Bypass UAC | Privilege Escalation, Defense Evasion | ✅ | ✅ | Atomic Red Team (eventvwr.exe) | Wazuh Rules 594 e 750 após syscheck forçado; mapeado para Defense Evasion + Impact | TLOA-Lab-Exercises |
| T1003.001 | OS Credential Dumping — LSASS Memory | Credential Access | ✅ | ❌ | Atomic Red Team (comsvcs.dll) | Gap crítico — nenhuma regra nativa no Wazuh; Sysmon EID 10 necessário com filtro por processos não autorizados acessando lsass.exe | TLOA-Lab-Exercises |
| T1547.001 | Boot or Logon Autostart Execution — Registry Run Keys | Persistence | ✅ | ⚠️ | Manual | Detectado via syscheck; mesma limitação de latência do T1112 | TLOA-IR-Cases (Case 001) |
| T1078 | Valid Accounts | Defense Evasion, Persistence, Initial Access | ✅ | ❌ | Netexec / Evil-WinRM | Autenticação com credenciais válidas comprometidas não gera alerta por padrão no Wazuh sem tuning de regras | TLOA-IR-Cases (Case 002) |
| T1558.003 | Steal or Forge Kerberos Tickets — Kerberoasting | Credential Access | ✅ | 🚫 | Rubeus / manual | Windows Server 2025 depreca RC4 por padrão — tickets AES-256 requerem hashcat com wordlist específica; documentado como hardening finding | TLOA-Lab-Exercises |
| T1550.002 | Use Alternate Authentication Material — Pass-the-Hash | Defense Evasion, Lateral Movement | ✅ | ❌ | Mimikatz + Netexec | DC01 bloqueou via SMB signing enforçado; WS01 (Windows 10) foi explorado com sucesso; Wazuh zero alertas em ambos os casos | TLOA-IR-Cases (Case 004) |
| T1021.006 | Remote Services — Windows Remote Management | Lateral Movement | ✅ | ❌ | Evil-WinRM | Sessão WinRM estabelecida com sucesso em WS01 via hash NTLM; Wazuh não gerou alertas; gap em autenticação lateral sem regras customizadas | TLOA-IR-Cases (Case 004) |

---

## Resumo por Tática ATT&CK

| Tática | Técnicas cobertas | Detectadas | Com gap |
|---|---|---|---|
| Reconnaissance | T1046 | 0 | 1 |
| Persistence | T1053.005, T1547.001, T1078 | 1 | 2 |
| Privilege Escalation | T1053.005, T1548.002 | 2 | 0 |
| Defense Evasion | T1112, T1548.002, T1078, T1550.002 | 1 | 3 |
| Credential Access | T1003.001, T1558.003 | 0 | 1 + 1 bloqueado |
| Lateral Movement | T1550.002, T1021.006 | 0 | 2 |

**Total de técnicas executadas:** 10
**Detectadas em tempo real:** 2 (T1053.005, T1548.002)
**Detectadas com latência:** 2 (T1112, T1547.001)
**Gaps confirmados:** 5
**Bloqueadas por hardening:** 1 (T1558.003 — RC4 deprecation no WS2025)

---

## Análise dos Gaps por Causa Raiz

### Ausência de regras nativas

O Wazuh não possui regras nativas para algumas das técnicas mais críticas. Nesses casos, a detecção depende de criação de regras customizadas baseadas em eventos Sysmon.

| Técnica | Sysmon EID relevante | Ação recomendada |
|---|---|---|
| T1003.001 (LSASS dump) | EID 10 (ProcessAccess) | Regra customizada filtrando acesso ao lsass.exe por processos fora de whitelist |
| T1550.002 (PTH) | EID 3 (NetworkConnect) + Windows Event 4624 Logon Type 3 | Correlação entre logon remoto e ausência de logon interativo prévio |
| T1021.006 (WinRM) | EID 3 (NetworkConnect) na porta 5985/5986 | Alerta em conexões WinRM originadas de estações de trabalho |

### Visibilidade de camada de rede ausente

Técnicas de reconhecimento e movimentação de rede não geram eventos no endpoint. O Wazuh, por ser um SIEM focado em telemetria de host, não cobre esse vetor sem complemento.

| Técnica | Gap | Solução |
|---|---|---|
| T1046 (Network recon) | Firewall bloqueia mas não gera log no Wazuh | Suricata ou Zeek para IDS de rede |
| T1021.006 (WinRM lateral) | Tráfego não monitorado na camada de rede | Captura de tráfego + regras de IDS para porta 5985 |

### Latência do File Integrity Monitoring (syscheck)

O ciclo padrão do syscheck no Wazuh é de 12 horas — adequado para auditoria periódica, mas insuficiente para detecção em tempo real de modificações de registro.

| Técnica | Solução |
|---|---|
| T1112 (Modify Registry) | Sysmon EID 12 (RegistryEvent — Object create/delete) + EID 13 (RegistryEvent — Value set) |
| T1547.001 (Registry Run Keys) | Mesma abordagem — Sysmon EID 13 com filtro em chaves de Run/RunOnce |

---

## Hardening Findings — Windows Server 2025

O Windows Server 2025 introduz comportamentos de hardening por padrão que impactam técnicas clássicas. Esses comportamentos não foram contornados — foram documentados como findings técnicos realistas.

| Restrição | Técnica impactada | Comportamento observado |
|---|---|---|
| RC4 deprecation | T1558.003 (Kerberoasting) | Tickets emitidos apenas em AES-256; RC4 bloqueado; Rubeus falhou na solicitação de ticket RC4 |
| SMB signing enforçado | T1550.002 (Pass-the-Hash via SMB) | Netexec rejeitado pelo DC01 em autenticação PTH via SMB; WS01 (Windows 10, sem SMB signing obrigatório) foi explorado com sucesso |
| LDAP signing | Enumeração AD via LDAP não assinado | Ferramentas de enumeração que não suportam LDAP signing falham contra DC01 |

---

## Próximas Técnicas a Validar

| ATT&CK ID | Técnica | Tática | Prioridade | Justificativa |
|---|---|---|---|---|
| T1055 | Process Injection | Defense Evasion, Privilege Escalation | 🔴 Alta | Técnica core de APTs e ransomware; ausência de EDR torna gap esperado |
| T1059.001 | PowerShell | Execution | 🔴 Alta | Vetor de execução mais comum; cobertura de logging precisa ser validada |
| T1136.001 | Create Local Account | Persistence | 🟡 Média | Técnica simples com boa cobertura esperada via Windows Event 4720 |
| T1069 | Permission Groups Discovery | Discovery | 🟡 Média | Enumeração de grupos AD — cobertura parcial esperada via LDAP logging |
| T1070.001 | Indicator Removal — Clear Windows Event Logs | Defense Evasion | 🔴 Alta | Técnica de anti-forense; impacto direto na capacidade de investigação |

---

*Atualizado conforme novos exercícios e IR Cases são executados. Cada entrada referencia o repositório de origem com a documentação técnica completa.*
