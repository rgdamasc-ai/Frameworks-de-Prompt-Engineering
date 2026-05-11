# Questão 01 — Dockerfile de produção para a API Lift

**Data:** 2026-05-11
**Ferramenta usada:** Claude Code
**Modelo:** claude-sonnet-4-6
**Framework:** R-T-F

---

## Contexto

> Containerização de uma API Python/Flask chamada "Lift" para deploy em Kubernetes.
> O objetivo é criar um Dockerfile de produção seguindo boas práticas de segurança, eficiência de camadas e conformidade com ambientes produtivos.

---

## Prompt usado

```
# Role
Você é um engenheiro DevOps sênior especialista em containerização de aplicações,
com profundo conhecimento em Docker, boas práticas de segurança para imagens
de produção e deploy em clusters Kubernetes.

# Task
Crie um Dockerfile de produção para containerizar uma API Python/Flask chamada
"Lift", que será deployada em um cluster Kubernetes.

Requisitos obrigatórios:
- Linguagem/framework: Python com Flask, servida via Gunicorn
- Porta exposta: 8080
- Comando de inicialização: gunicorn --bind 0.0.0.0:8080 --workers 4 app:app
- Dependências: instaladas a partir do arquivo requirements.txt
- Variáveis de ambiente obrigatórias em runtime: DATABASE_URL e API_KEY
  (devem estar declaradas com ENV, mas sem valores hardcoded)
- Estrutura do projeto:
    lift/
    ├── app.py
    ├── requirements.txt
    ├── lib/
    │   ├── auth.py
    │   └── storage.py
    └── tests/
        └── test_app.py

Boas práticas que DEVEM ser aplicadas:
1. Usar imagem base oficial slim (ex: python:3.12-slim) para reduzir superfície de ataque
2. Multistage build se trouxer benefício real para este caso
3. Criar e usar usuário não-root para executar a aplicação
4. Copiar apenas os arquivos necessários para produção (excluir tests/)
5. Separar o COPY do requirements.txt do COPY do restante do código,
   para aproveitar o cache de camadas do Docker
6. Usar WORKDIR adequado
7. Adicionar LABEL com metadados relevantes (maintainer, versão, descrição)
8. Garantir que pip não salve cache desnecessário (--no-cache-dir)
9. Declarar EXPOSE para documentar a porta
10. Usar CMD (não ENTRYPOINT) para o comando Gunicorn,
    permitindo override fácil no Kubernetes

# Format
Retorne EXCLUSIVAMENTE:

1. O Dockerfile completo, pronto para uso em produção, com comentários
   explicativos em cada bloco de instrução (FROM, RUN, COPY, ENV, etc.)

2. Um arquivo .dockerignore recomendado para este projeto

3. Uma tabela markdown com os 5 principais comandos Docker para
   build, execução local e inspeção da imagem gerada:

   | Comando | Descrição |
   |---------|-----------|
   | ...     | ...       |

Não inclua introduções, conclusões ou explicações fora das estruturas acima.
Use blocos de código com syntax highlighting adequado (dockerfile, plaintext).
```

---

## Output completo da IA

**1. Dockerfile**

```dockerfile
# ==============================================================================
# Stage 1: builder — instala dependências (inclui ferramentas de compilação)
# ==============================================================================
FROM python:3.12-slim AS builder

# Diretório de trabalho isolado para o estágio de build
WORKDIR /build

# Copia requirements.txt ANTES do código-fonte para aproveitar o cache de
# camadas: se apenas o código mudar, esta camada não é reconstruída
COPY requirements.txt .

# Instala dependências em prefixo isolado; --no-cache-dir evita armazenar
# cache do pip, mantendo a camada enxuta
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

# ==============================================================================
# Stage 2: runtime — imagem final mínima, sem ferramentas de build
# O multistage garante que gcc, pip e outros artefatos de compilação
# não existam na imagem que chegará ao cluster Kubernetes
# ==============================================================================
FROM python:3.12-slim

# Metadados OCI para rastreabilidade, auditoria e inventário de imagens
LABEL maintainer="lift-team@company.com" \
      version="1.0.0" \
      description="API Lift — Python/Flask servida via Gunicorn"

# Variáveis obrigatórias declaradas sem valores hardcoded;
# devem ser injetadas via Kubernetes Secret ou ConfigMap no deploy
ENV DATABASE_URL="" \
    API_KEY=""

# Diretório raiz da aplicação dentro do container
WORKDIR /app

# Copia as dependências já instaladas do stage builder para o runtime final;
# apenas os pacotes — não o pip nem as ferramentas de build
COPY --from=builder /install /usr/local

# Copia requirements.txt separadamente (antes do código) para preservar cache:
# rebuild de dependências só ocorre quando requirements.txt muda
COPY requirements.txt .

# Copia apenas os arquivos necessários para produção;
# tests/ é deliberadamente excluído desta etapa de COPY
COPY app.py .
COPY lib/ ./lib/

# Cria grupo e usuário não-root dedicados, sem home nem shell interativo,
# seguindo o princípio de menor privilégio para execução em produção
RUN addgroup --system liftgroup \
    && adduser --system --ingroup liftgroup --no-create-home --shell /bin/false liftuser \
    && chown -R liftuser:liftgroup /app

# Troca para o usuário não-root antes de iniciar qualquer processo de rede
USER liftuser

# Documenta a porta da aplicação (não realiza bind efetivo —
# o bind ocorre no CMD e no Service do Kubernetes)
EXPOSE 8080

# CMD (não ENTRYPOINT) permite override fácil no Kubernetes:
# initContainers, liveness probes e kubectl exec podem sobrescrever
# sem necessidade do flag --entrypoint
CMD ["gunicorn", "--bind", "0.0.0.0:8080", "--workers", "4", "app:app"]
```

---

**2. .dockerignore**

```plaintext
# Controle de versão
.git/
.gitignore

# Testes e artefatos de cobertura (não vão para produção)
tests/
.pytest_cache/
htmlcov/
.coverage

# Artefatos de compilação Python
__pycache__/
*.py[cod]
*.pyo
*.pyd
*.egg-info/
dist/
build/

# Ambientes virtuais locais
.venv/
venv/
env/

# IDE e editores
.vscode/
.idea/
*.swp
*.swo
.DS_Store

# Arquivos Docker e de orquestração (não necessários dentro da imagem)
Dockerfile*
docker-compose*.yml
.dockerignore

# Documentação
*.md
docs/

# Variáveis de ambiente locais — NUNCA devem entrar na imagem
.env
.env.*
*.env
```

---

**3. Principais comandos Docker**

| Comando | Descrição |
|---------|-----------|
| `docker build -t lift-api:1.0.0 .` | Constrói a imagem com tag versionada a partir do Dockerfile |
| `docker run --rm -p 8080:8080 -e DATABASE_URL=postgresql://localhost/lift -e API_KEY=secret lift-api:1.0.0` | Executa o container localmente injetando as variáveis de runtime obrigatórias |
| `docker inspect lift-api:1.0.0` | Exibe metadados completos da imagem: camadas, ENV, LABELs, usuário e ponto de entrada |
| `docker history --no-trunc lift-api:1.0.0` | Mostra o histórico de camadas com tamanho de cada instrução para análise de otimização |
| `docker image ls lift-api` | Lista todas as tags da imagem Lift com tamanho total e data de criação |

---

## Análise do framework

**Justificativa:** O prompt segue o framework **R-T-F** (Role / Task / Format). Cada elemento do acrônimo está explicitamente delimitado por um cabeçalho de seção (`# Role`, `# Task`, `# Format`), tornando a estrutura transparente e fácil de auditar.

### Mapeamento

| Elemento | Trecho do prompt |
|----------|-----------------|
| **R** — Role | `"Você é um engenheiro DevOps sênior especialista em containerização de aplicações, com profundo conhecimento em Docker, boas práticas de segurança para imagens de produção e deploy em clusters Kubernetes."` |
| **T** — Task | `"Crie um Dockerfile de produção para containerizar uma API Python/Flask chamada 'Lift'... Requisitos obrigatórios: ... Boas práticas que DEVEM ser aplicadas: ..."` |
| **F** — Format | `"Retorne EXCLUSIVAMENTE: 1. O Dockerfile completo... 2. Um arquivo .dockerignore... 3. Uma tabela markdown..."` |
