
---

# Documento 2: `docs/REQUISITOS_E_ARQUITETURA.md` (Documento de Engenharia)

```markdown
# Especificação de Requisitos de Software e Arquitetura do Sistema (SRS)

**Projeto:** ControlFrota SaaS  
**Versão:** 2.4.0  
**Autor:** Romulo Valentim  
**Público-alvo:** Engenheiros de Software, Tech Leads e Avaliadores Técnicos  

---

## 1. Visão Geral e Escopo
O ControlFrota é uma plataforma B2B concebida sob o modelo de Software como Serviço (SaaS) multi-inquilino (multi-tenant), focada na digitalização do ciclo de vida operacional de locadoras de veículos. 

O sistema substitui planilhas e formulários despadronizados por fluxos transacionais auditáveis, cobrindo:
1. Gestão contratual e cobrança recorrente em ciclos semanais.
2. Controle de integridade física dos veículos por meio de vistorias remotas com prova de presença.
3. Prevenção a perdas através de triagem pericial de CNHs e análise de inadimplência.

---

## 2. Requisitos Não-Funcionais (RNF)

| Identificador | Categoria | Descrição Técnica |
| :--- | :--- | :--- |
| **RNF-01** | **Segurança & Multi-tenancy** | Isolamento lógico rigoroso de dados através de PostgreSQL Row Level Security (RLS). Nenhuma query na camada de aplicação tem acesso a registros de outro `tenant_id`. |
| **RNF-02** | **Confiabilidade Financeira** | Processamento de webhooks com idempotência garantida através de controle transacional de concorrência (`durable locks`), impedindo corrida em callbacks de liquidação. |
| **RNF-03** | **Conformidade LGPD** | Anonimização e aplicação de máscaras em dados de pessoas físicas em listagens públicas/compartilhadas. Retenção estrita de fotos periciais para otimização de custo e minimização de dados. |
| **RNF-04** | **Desempenho & Edge** | Renderização híbrida (SSR e hidratação seletiva) garantindo carregamento de dashboards analíticos com First Contentful Paint (FCP) inferior a 1,2s. |
| **RNF-05** | **Integridade de Mídia** | Armazenamento de imagens periciais em storage privado, servidas via URLs pré-assinadas com tempo de expiração curto para evitar vazamento de dados de localização e motorista. |

---

## 3. Requisitos Funcionais (RF) e Regras de Negócio Críticas

### Módulo 1: Gestão de Contratos e Ciclos de Cobrança Semanal

- **RF-01: Geração Idempotente de Cobranças:** O sistema calcula as cobranças semanais com base no `dia_semana` estipulado no contrato. A rotina é estritamente idempotente: múltiplos disparos da mesma rotina de sincronização não duplicam faturas para o mesmo ciclo de vigência.
- **RF-02: Isolamento de Taxa de Adesão:** A taxa de adesão (setup) do contrato é tratada como entidade financeira independente do depósito caução. O valor não pode ser abatido de despesas operacionais nem sofrer devolução automática no distrato.
- **RF-03: Bloqueio de Segurança para Clientes Inadimplentes:** Quando um contrato atinge mais de 3 dias de atraso na fatura semanal, o sistema emite alerta operacional e sinaliza o veículo como apto para solicitação de recolhimento ou corte de despacho.

### Módulo 2: Perícia Remota (Self-Vistoria Semanal)

- **RF-04: Protocolo de Captura de 9 Ângulos:** Para validar a integridade do bem, o motorista é obrigado a registrar:
  1. Frente
  2. Lateral Direita
  3. Traseira
  4. Lateral Esquerda
  5. Teto
  6. Painel de Instrumentos (Odômetro + Luzes de Alerta)
  7. Pneu Dianteiro Esquerdo
  8. Pneu Dianteiro Direito
  9. Pneus Traseiros
- **RF-05: Prova de Presença (Anti-Spoofing de Localização):** Cada foto capturada exige metadados de geolocalização nativos do dispositivo no instante exato do clique. Imagens carregadas do rolo de câmera sem dados GPS de tempo real são sumariamente rejeitadas.
- **RF-06: Regra de Atualização de Quilometragem Oficial:** O leitor de odômetro sugere a quilometragem a partir da foto do painel, mas o valor só é promovido a **KM Oficial do Veículo** após conferência e aprovação explícita do locador. A quilometragem é estritamente não-decrescente.
- **RF-07: Fluxo de Reprovação de Laudo:** Caso o locador recuse o laudo (ex: foto borrada ou hodômetro ilegível), as fotos da vistoria rejeitada são purgadas do storage de longa duração, um novo link de vistoria pendente é gerado imediatamente e notificado via mensageria ao condutor.

### Módulo 3: Análise Pericial de CNH e Prevenção a Fraudes

- **RF-08: Distinção de Emissão vs. Validade:** No modelo CNH brasileiro (Resolução CONTRAN 886/2021), o algoritmo diferencia obrigatoriamente o Campo 4a (Data de Emissão da via física) do Campo 4b (Data de Validade jurídica do exame de aptidão física e mental).
- **RF-09: Regra Antifraude PPD + EAR:** De acordo com o Código de Trânsito Brasileiro, a Permissão para Dirigir (PPD) tem validade improrrogável de exatamente 1 ano e é juridicamente incompatível com o Exercício de Atividade Remunerada (EAR). Caso um documento apresente período de 1 ano com a anotação "EAR", o sistema classifica o condutor com alerta vermelho de **Fraude Documental por Falsificação**.
- **RF-10: Âncoras Geográficas Cumulativas:** A análise de risco não pode invalidar um motorista por divergência entre o estado emissor do CPF e o estado da CNH. O sistema calcula a compatibilidade territorial por âncoras cumulativas (naturalidade, UF do CPF, UF de emissão da CNH e comprovante de endereço atual).

### Módulo 4: Automação Operacional Multi-Canal

- **RF-11: Protocolo de Confirmação para Operações Destrutivas:** O assistente operacional via WhatsApp e Web permite consultas e mutações. Para qualquer mutação na base (como registro de manutenção ou alteração de odômetro), o sistema exige confirmação explícita de dois passos (SIM/NÃO) do gestor autorizado, registrando o ID da ação e payload em log de auditoria antes da execução no banco de dados.

---

## 4. Diagrama de Estados do Veículo

```text
       [Cadastrado]
            |
            v
     [Disponível] <-----------------------+
       |        ^                          |
       |        | (Devolução / Vistoria OK)|
       v        |                          |
    [Locado] ---+                          |
       |                                   |
       | (Alerta de KM / Troca de Óleo)    |
       v                                   |
  [Manutenção] ----------------------------+
