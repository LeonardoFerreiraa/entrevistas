# Entrevistas — estudo com Claude Code

Repositório para versionar **entrevistas técnicas** conduzidas entre mim e o Claude Code.

O Claude assume o papel de **entrevistador técnico sênior** e conduz uma dinâmica de entrevista real (uma pergunta por vez, escalando do básico ao avançado, cobrando nomenclatura canônica e o "porquê"). O objetivo é estudo e prática.

## Arquivos

| Arquivo | Descrição |
|---|---|
| [`prompt.md`](prompt.md) | Instrução inicial que define o papel do entrevistador, as regras da dinâmica e o formato de saída. Cole no início da sessão. |
| [`entrevista-staff-backend-java.md`](entrevista-staff-backend-java.md) | Sessão de entrevista — perguntas, respostas e observações do entrevistador (Staff Backend Java). |

## Como funciona

1. Forneça o [`prompt.md`](prompt.md) como instrução inicial e edite as seções **Configuração** e **Temas**.
2. O modelo conduz a entrevista, uma pergunta por vez, registrando em dois arquivos:
   - **entrevista** — só pergunta + resposta (sem spoiler);
   - **avaliação** — gabarito + observações críticas (revelado no fim).
3. Ao cobrir todos os temas, gera um resumo final com veredito, teto por tema e o que estudar a seguir.

Veja os detalhes da dinâmica em [`prompt.md`](prompt.md).
