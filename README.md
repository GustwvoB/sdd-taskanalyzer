# sdd-taskanalyzer

[![CI](https://github.com/GustwvoB/sdd-taskanalyzer/actions/workflows/ci.yml/badge.svg?branch=main)](https://github.com/GustwvoB/sdd-taskanalyzer/actions/workflows/ci.yml)

Módulo Python que analisa uma lista de tarefas e calcula métricas de produtividade: total de tarefas por situação, tempo médio de conclusão (geral e por prioridade) e taxa de atraso.

Este projeto é a Fase 2 de um desafio do Bootcamp III (Análise e Desenvolvimento de Sistemas, UniCEUB). Ele foi feito com **Spec-Driven Development (SDD)**: primeiro vieram a especificação e os testes, depois o código, que foi gerado por uma IA e revisado por mim.

## Status do build

- CI: o GitHub Actions roda o Test Harness a cada push ou Pull Request nas branches `main` e `develop` (veja o selo no topo desta página)
- Testes: **4 passed** (`python -m pytest -v`)
- Ambientes testados: Python 3.14.3 (local) e Python 3.11 (CI e Docker)
- Versão mínima exigida: Python 3.11+

## Como o projeto foi feito

1. **Especificação:** o contrato do módulo está em `specs/task_analyzer_spec.md`.
2. **Regras para a IA:** as diretrizes e as proibições estão em `CONTEXT_RULES.md`.
3. **Testes primeiro:** o Test Harness (`tests/test_harness.py`) foi escrito antes do código.
4. **Código com IA:** o `src/task_analyzer.py` foi gerado com Gemini, que recebeu a especificação e as regras como contexto.
5. **Homologação humana:** revisei o código contra as regras, encontrei problemas, pedi a correção e só depois fiz o commit.
6. **Automação (Fase 3):** um pipeline do GitHub Actions e um container Docker rodam os testes automaticamente, em ambiente limpo e com Python 3.11.

Cada etapa virou uma branch `feature/...` e um Pull Request para a `main`.

## Estrutura do repositório

```
sdd-taskanalyzer/
├── .github/
│   └── workflows/
│       └── ci.yml
├── Dockerfile
├── docker-compose.yml
├── .dockerignore
├── CONTEXT_RULES.md
├── README.md
├── .gitignore
├── requirements.txt
├── specs/
│   └── task_analyzer_spec.md
├── tests/
│   └── test_harness.py
└── src/
    └── task_analyzer.py
```

## Como instalar

Você precisa do Python 3.11 ou superior.

```
git clone https://github.com/GustwvoB/sdd-taskanalyzer.git
cd sdd-taskanalyzer
python -m venv venv
venv\Scripts\activate
pip install -r requirements.txt
```

No Linux ou no Mac, ative o ambiente com `source venv/bin/activate`.

## Como rodar os testes

Na raiz do projeto, com o ambiente virtual ativado:

```
python -m pytest -v
```

Use `python -m pytest` (e não só `pytest`) para o Python conseguir enxergar a pasta `src`.

| Teste | O que valida |
|---|---|
| `test_cenario_1_sucesso` | Totais, tempo médio (geral e por prioridade) e taxa de atraso |
| `test_cenario_2_data_conclusao_anterior_a_criacao` | Dispara `TaskValidationError` para datas inconsistentes |
| `test_borda_lista_vazia` | Lista vazia retorna zeros, sem exceção |
| `test_borda_apenas_pendentes` | Só tarefas pendentes retornam `0.0` nas médias, sem exceção |

## Como rodar com Docker

Não precisa instalar o Python nem criar ambiente virtual, só ter o Docker instalado e aberto. Na raiz do projeto:

```
docker compose run --rm testes
```

O container usa Python 3.11, instala o `requirements.txt` e roda o Test Harness. A saída deve terminar com `4 passed`.

Também dá para fazer em dois passos:

```
docker build -t sdd-taskanalyzer .
docker run --rm sdd-taskanalyzer
```

## Integração contínua (CI)

O arquivo `.github/workflows/ci.yml` define o pipeline do GitHub Actions:

- **Gatilhos:** push e Pull Request nas branches `main` e `develop`
- **Ambiente:** `ubuntu-latest` com Python 3.11
- **Passos:** baixar o código, instalar as dependências do `requirements.txt` e rodar `python -m pytest tests/test_harness.py -v`

Se algum teste falhar, o pipeline fica vermelho e o problema aparece na aba **Checks** do Pull Request.

## Exemplo de uso

```python
from datetime import datetime, timezone
from src.task_analyzer import analyze_tasks

tarefas = [
    {
        "id_tarefa": 1,
        "titulo": "Escrever relatório",
        "prioridade": "alta",
        "status": "concluida",
        "data_criacao": datetime(2026, 9, 1, 8, tzinfo=timezone.utc),
        "data_inicio": None,
        "data_conclusao": datetime(2026, 9, 1, 10, tzinfo=timezone.utc),
        "prazo": datetime(2026, 9, 1, 12, tzinfo=timezone.utc),
    }
]

print(analyze_tasks(tarefas))
```

Resultado:

```
{'total_tarefas': 1, 'total_concluidas': 1, 'total_pendentes': 0, 'tempo_medio_conclusao_horas': 2.0, 'tempo_medio_conclusao_por_prioridade_horas': {'baixa': 0.0, 'media': 0.0, 'alta': 2.0}, 'taxa_atraso_percentual': 0.0}
```

O módulo também escreve logs informativos no terminal, por isso podem aparecer linhas extras antes do resultado.

## Regras de negócio resumidas

- O tempo de conclusão é `data_conclusao` menos `data_criacao`, em horas.
- Uma tarefa está atrasada quando foi concluída **depois** do prazo (concluir exatamente no prazo não é atraso).
- Só tarefas concluídas entram nas médias e na taxa de atraso. Pendentes contam apenas em `total_pendentes` e canceladas apenas em `total_tarefas`.
- Se não houver tarefas concluídas, as médias e a taxa valem `0.0`, sem erro.
- Dados inválidos disparam `TaskValidationError`, com o id da tarefa na mensagem.

Os detalhes completos estão em `specs/task_analyzer_spec.md`.

## Limitações conhecidas

- Todas as datas precisam ter fuso horário (UTC). Misturar datas com e sem fuso gera um `TypeError` do próprio Python, porque a especificação ainda não define esse caso.
- O módulo é *stateless*: não salva nada em arquivo nem em banco de dados.
