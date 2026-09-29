# teste_analista
# Projeto de Implantação de Folha de Pagamento - Estado de Goiás (VOBYS)
## Especificação Técnica: Regras Lógicas de Lançamento de Rubricas Compulsórias


1. Agrupamento dos Regimes de Trabalho conforme (ANEXO I), informado nas regras do teste.
O Anexo I lista seis categorias funcionais. Para fins de gestão previdenciária, contabilidade pública e apuração fiscal perante a Receita Federal.

Grupo 1: RPPS 
ESTATUTÁRIO
SERVIDOR PÚBLICO CIVIL - GOIASPREV (Regime Próprio Estadual - Fundo Financeiro/Previdenciário Civil) | Tributação progressiva na fonte (CF/88, Art. 157, I).

Grupo 2: Proteção Social Militar
SERVIDOR PÚBLICO MILITAR - SPSM / GOIASPREV Militar (Fundo de Proteção Social dos Militares de Goiás) | Tributação progressiva na fonte (CF/88, Art. 157, I).

Grupo 3: RGPS Geral
SERVIDOR COMISSIONADO
TEMPORÁRIO
PRESTADOR DE SERVIÇOS - INSS (Regime Geral de Previdência Social / Guia DCTFWeb) | Tributação progressiva na fonte (CF/88, Art. 157, I) .

- Obs: As linhas ESTATUTÁRIO (servidores submetidos à Lei Estadual nº 20.756/2019) e SERVIDOR PÚBLICO CIVIL (carreiras de Estado com estatutos específicos, ex.: Magistério, Fisco, Polícia Civil) compartilham a mesma natureza jurídica estatutária e subordinação previdenciária à GOIASPREV, integrando conjuntamente o Grupo 1.


2. Matriz de Rubricas de Desconto Sugeridas
Para garantir a segregação de receitas públicas, atender às exigências de conformidade dos eventos de tabela do eSocial (S-1010) e evitar rejeições nas folhas mensais (S-1200), foram definidas quatro rubricas de desconto compulsório:


Código Sugerido  -  Descrição da Rubrica   -    Tipo    -  Incidência eSocial (codIncCP)   -   Finalidade e Âmbito de Aplicação
DESC_RPPS_CIV       Contribuição 
                    Previdenciária - RPPS     Desconto     31 (Previdência Própria - RPPS)     Servidores civis estatutários 
                    Civil                                                                      ativos, bem como aposentados e 
                                                                                               pensionistas que percebam 
                                                                                               proventos acima do teto do RGPS.
                                                                    

DESC_RPPS_MIL      Contribuição Proteção      Desconto     32 (Pensão Militar / SPSM)         Militares da ativa, da reserva                         Social / Pensão Militar                                                    remunerada, reformados e                                                                                                          pensionistas militares do Estado de 
                                                                                              Goiás.


                        
DESC_INSS_RGPS    Contribuição Previdenciária  Desconto   11 (Previdência Geral - RGPS)       Comissionados sem vínculo efetivo, 
                  RGPS / INSS                                                                 temporários (Lei 20.918/20) e
                                                                                              prestadores autônomos que não
                                                                                              comprovaram recolhimento integral
                                                                                              em outra fonte.


DESC_IRRF_FOLHA    Imposto de Renda            Desconto   00 (Isento de CP) / IRRF: 01        Retenção tributária aplicável a 
                    Retido na Fonte - Folha                                                   todos os vínculos com base
                                                                                              tributável positiva, ressalvadas
                                                                                              hipóteses de isenção legal/judicial.
                  



3. Critérios de Lógica para Lançamentos
Apenas os vínculos que retornarem status VERDADEIRO recebem a inserção da rubrica no contracheque da competência.

Rubrica: DESC_RPPS_CIV - O objetivo é aplicar a previdência civil estadual aos estatutários em atividade e restringir a retenção de inativos apenas aos proventos que superem o limite máximo do RGPS, conforme (Art. 40, § 18 da CF/88 e LC Estadual nº 161/2020).

SE (Regime_Trabalho EM ["ESTATUTARIO", "SERVIDOR_PUBLICO_CIVIL"])
E (Data_Inicio_Vinculo <= Data_Fim_Competencia)
E (Data_Fim_Vinculo IS NULL OU Data_Fim_Vinculo >= Data_Inicio_Competencia)
ENTÃO:
    SE (Situacao_Funcional == "ATIVO") ENTÃO:
        Lança_Rubrica(DESC_RPPS_CIV) = VERDADEIRO
    SENÃO SE (Situacao_Funcional EM ["APOSENTADO", "PENSIONISTA"]) ENTÃO:
        SE (Proventos_Brutos_Previdenciarios > Teto_Maximo_RGPS) ENTÃO:
            Lança_Rubrica(DESC_RPPS_CIV) = VERDADEIRO
        SENÃO:
            Lança_Rubrica(DESC_RPPS_CIV) = FALSO
    SENÃO:
        Lança_Rubrica(DESC_RPPS_CIV) = FALSO
SENÃO:
    Lança_Rubrica(DESC_RPPS_CIV) = FALSO




Rubrica: DESC_RPPS_MIL - o objetivo é descontar a alíquota uniforme do Sistema de Proteção Social dos Militares (Lei Federal nº 13.954/2019) sobre a remuneração integral de militares estaduais da ativa, reserva e reformados.

SE (Regime_Trabalho == "SERVIDOR_PUBLICO_MILITAR")
E (Situacao_Funcional EM ["ATIVO", "RESERVA_REMUNERADA", "REFORMADO", "PENSIONISTA"])
E (Data_Inicio_Vinculo <= Data_Fim_Competencia)
E (Data_Fim_Vinculo IS NULL OU Data_Fim_Vinculo >= Data_Inicio_Competencia)
ENTÃO:
    Lança_Rubrica(DESC_RPPS_MIL) = VERDADEIRO
SENÃO:
    Lança_Rubrica(DESC_RPPS_MIL) = FALSO



Rubrica: DESC_INSS_RGPS - o objetivo é Vincular comissionados, temporários e prestadores de serviços ao RGPS (Art. 40, § 13 da CF/88), implementando salvaguarda para evitar retenção indébita sobre prestadores autônomos que já recolheram sobre o teto previdenciário em outra fonte pagadora.

SE (Regime_Trabalho EM ["SERVIDOR_COMISSIONADO", "TEMPORARIO", "PRESTADOR_SERVICOS"])
E (Data_Inicio_Vinculo <= Data_Fim_Competencia)
E (Data_Fim_Vinculo IS NULL OU Data_Fim_Vinculo >= Data_Inicio_Competencia)
ENTÃO:
    SE (Regime_Trabalho == "PRESTADOR_SERVICOS" E Comprovou_Recolhimento_Teto_Outra_Fonte == VERDADEIRO) ENTÃO:
        Lança_Rubrica(DESC_INSS_RGPS) = FALSO
    SENÃO:
        Lança_Rubrica(DESC_INSS_RGPS) = VERDADEIRO
SENÃO:
    Lança_Rubrica(DESC_INSS_RGPS) = FALSO



Rubrica: DESC_IRRF_FOLHA - o objetivo é processar a retenção tributária federal (IRRF), priorizando a verificação cadastral de isenção médica (portadores de moléstia grave periciados pela Junta Médica Oficial - Lei Federal nº 7.713/1988) e ordens judiciais, antes da avaliação do limite de isenção da tabela mensal.

SE (Data_Inicio_Vinculo <= Data_Fim_Competencia)
E (Data_Fim_Vinculo IS NULL OU Data_Fim_Vinculo >= Data_Inicio_Competencia)
ENTÃO:
    SE (Indicador_Isencao_Molestia_Grave == VERDADEIRO OU Indicador_Isencao_Judicial == VERDADEIRO) ENTÃO:
        Lança_Rubrica(DESC_IRRF_FOLHA) = FALSO
    SENÃO SE (Rendimento_Tributavel_Apurado > Limite_Isencao_Tabela_RFB) ENTÃO:
        Lança_Rubrica(DESC_IRRF_FOLHA) = VERDADEIRO
    SENÃO:
        Lança_Rubrica(DESC_IRRF_FOLHA) = FALSO
SENÃO:
    Lança_Rubrica(DESC_IRRF_FOLHA) = FALSO








