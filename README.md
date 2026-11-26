<p align="center">
  <a href="https://ag.tavardt.com/br/">
    <img src="https://ag.tavardt.com/img/tavardt-banner.png" alt="TAVARDT" width="1024" />
  </a>
</p>

# Auditor de Segurança CI/CD e Hardening da Cadeia de Suprimentos de Software

[![Live Demo](https://img.shields.io/badge/Live%20Demo-GitHub%20Pages-5b3cf5.svg)](https://tavardt-brasil.github.io/ci-cd-security-auditor-br/)
[![Python](https://img.shields.io/badge/Python-3.8%2B-blue.svg)](https://www.python.org/)
[![Standards](https://img.shields.io/badge/Standards-CIS%20%7C%20OWASP%20CI%2FCD-blue.svg)](https://tavardt-brasil.github.io/ci-cd-security-auditor-br/)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

Guia corporativo de engenharia defensiva, especificações de imunização de esteiras de automação e utilitário determinístico de auditoria CLI para blindagem da cadeia de suprimentos de software em GitHub Actions e ecossistemas modernos de CI/CD.

***

## Panorama Executivo: O Cenário de Ameaças em CI/CD

As esteiras de Integração Contínua e Entrega Contínua (CI/CD) representam o coração da engenharia de software contemporânea. Por possuírem acesso direto à rede corporativa, permissões de escrita nos repositórios e credenciais críticas de deploy, os pipelines tornaram-se o alvo primordial de ataques avançados à cadeia de suprimentos.

Quando equipes de desenvolvimento priorizam a velocidade de entrega sem controles perimetrais de segurança, os fluxos de trabalho herdam vulnerabilidades estruturais críticas:
* Gatilhos de build executam código não confiável de forks dentro de ambientes privilegiados.
* Dependências externas e actions de terceiros utilizam tags mutáveis sujeitas a sequestro de contas.
* Tokens de execução herdam permissões globais excessivas de escrita em todo o repositório.
* Ambientes de execução (runners) operam sem filtros de rede na saída (egress), viabilizando a exfiltração de segredos.

Este repositório estabelece um padrão corporativo de defesa em profundidade. Ele combina modelos de configuração testados em produção com um analisador estático determinístico em Python (`audit_ci_cd_security.py`) para garantir conformidade estrita e zero telemetria.

***

## Superfícies Críticas de Ataque em Pipelines de Automação

### 1. O Risco Crítico do Gatilho `pull_request_target`
Workflows acionados pelo evento `pull_request_target` rodam no contexto do repositório base e herdam acesso direto aos segredos de produção:
* **Mecanismo da Falha:** Workflows que combinam esse gatilho com o checkout do commit do pull request (`github.event.pull_request.head.sha`) baixam código enviado por qualquer terceiro diretamente para o runner privilegiado.
* **Impacto Operacional:** Atacantes externos podem alterar scripts de compilação, executar comandos arbitrários no runner e capturar credenciais de deploy sem aprovação prévia.
* **Diretriz Defensiva:** Jamais execute checkout de código não confiável sob `pull_request_target`. Isole rotinas de validação em eventos tradicionais de `pull_request` restritos.

### 2. Actions de Terceiros sem Fixação por Commit SHA Imutável
Workflows que referenciam actions de terceiros por meio de tags de versão (como `@v3` ou `@main`) criam dependências upstream vulneráveis:
* **Mecanismo da Falha:** Tags no Git são ponteiros mutáveis que podem ser reescritos caso a conta do mantenedor da action seja comprometida.
* **Impacto Operacional:** Um commit malicioso inserido em uma action popular compromete instantaneamente milhares de esteiras corporativas simultâneas.
* **Diretriz Defensiva:** Fixe todas as actions externas obrigatoriamente pelo hash SHA de 40 caracteres, acompanhado de comentário com a versão correspondente.

### 3. Tokens com Permissões Excessivas (`permissions: write-all`)
Pipelines executam com frequência herdando o token padrão `GITHUB_TOKEN` com escopos permissivos definidos no nível da organização:
* **Mecanismo da Falha:** A omissão do bloco restritivo `permissions:` concede ao token permissões irrestritas de escrita sobre o repositório, pacotes e releases.
* **Impacto Operacional:** Dependências comprometidas executadas durante o build podem adulterar código em branches protegidas ou publicar artefatos adulterados.
* **Diretriz Defensiva:** Defina explicitamente `permissions: contents: read` no topo de cada manifesto de automação, limitando permissões de escrita exclusivamente a jobs específicos e isolados.

### 4. Conectividade Irrestrita de Saída (Egress) nos Runners
Runners de build convencionais operam com acesso irrestrito à internet:
* **Mecanismo da Falha:** Scripts maliciosos executados durante o build podem abrir conexões reversas e transmitir variáveis de ambiente para servidores externos de comando.
* **Diretriz Defensiva:** Aplique regras de filtragem de rede (egress filtering) no runner, permitindo apenas a comunicação com registros de pacotes e repositórios homologados.

***

## Blueprints de Mitigação Defensiva Homologados

O repositório fornece configurações de arquitetura prontas para implementação em ambientes corporativos:

### 1. Workflow Endurecido para GitHub Actions (`hardened-workflow.yml`)
Implementa o princípio do menor privilégio, fixação por commit SHA e isolamento de credenciais:

```yaml
name: Enterprise Hardened Pipeline

on:
  pull_request:
    branches: [main]

permissions:
  contents: read

jobs:
  audit-and-build:
    runs-on: ubuntu-latest
    steps:
      - name: Hardened Checkout
        uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11 # v4.1.1
        with:
          persist-credentials: false

      - name: Inicializacao Defensiva do Python
        uses: actions/setup-python@0a5c61591373683505ea898e09a3ea4f39ef2b9c # v5.0.0
        with:
          python-version: '3.11'
          cache: 'pip'

      - name: Execucao de Analise Estatica Deterministica
        run: |
          python audit_ci_cd_security.py .github/workflows/
```

### 2. Script de Fixação Automatizada de Actions (`action-pinning.sh`)
Varre manifestos de workflow e valida a conformidade de SHAs imutáveis:

```bash
#!/usr/bin/env bash
set -euo pipefail

# Inspeciona manifestos e assegura fixacao por hash de 40 caracteres
find .github/workflows -type f \( -name "*.yml" -o -name "*.yaml" \) | while read -r workflow; do
  echo "[*] Auditando workflow: ${workflow}"
  grep -E "uses: [a-zA-Z0-9_\-\.\/]+@[a-f0-9]{40}" "${workflow}" || echo "[!] Actions mutaveis detectadas"
done
```

### 3. Filtragem de Rede na Saída do Runner (`step-security-harden.yml`)
Bloqueia conexões externas não autorizadas partindo do ambiente de execução:

```yaml
- name: Endurecer Rede do Runner
  uses: step-security/harden-runner@63c24ba6bd7ba022e95695ff85de572c04a18142 # v2.7.0
  with:
    egress-policy: block
    allowed-endpoints: >
      github.com:443
      api.github.com:443
      pypi.org:443
      files.pythonhosted.org:443
```

***

## Ferramenta Oficial de Auditoria CLI

Ambientes de navegadores operam sob restrições de sandbox que impedem a leitura de sistemas de arquivos locais e a inspeção direta de configurações de infraestrutura. Para viabilizar auditorias técnicas reprodutíveis e com zero telemetria, a TAVARDT desenvolveu um analisador estático em Python concebido para terminais locais e esteiras de CI/CD.

### Mecanismo de Análise Estática

O utilitário (`audit_ci_cd_security.py`) executa quatro validações determinísticas:
* **Auditoria de Gatilhos:** Analisa declarações de eventos para sinalizar usos perigosos de `pull_request_target` combinados com checkout de código externo.
* **Verificação de Permissões Mínimas:** Identifica diretivas `write-all` ou a ausência de blocos restritivos de permissão.
* **Inspeção de Fixação de Dependências:** Valida cada declaração `uses:` via expressões regulares, assegurando a presença de hashes SHA de 40 dígitos.
* **Varredura de Segredos em Texto Claro:** Avalia blocos shell e variáveis de ambiente em busca de credenciais expostas sem máscara.

### Instruções de Execução

Baixe e execute a ferramenta diretamente contra os manifestos de automação do repositório:

```bash
# 1. Obter o script oficial
curl -sO https://raw.githubusercontent.com/TAVARDT-Brasil/ci-cd-security-auditor-br/main/audit_ci_cd_security.py

# 2. Executar analise estatica contra workflows locais
python audit_ci_cd_security.py .github/workflows/
```

***

## Perguntas Frequentes de Arquitetura (FAQ Técnico)

### 1. Por que o gatilho pull_request_target é classificado como uma das vulnerabilidades mais críticas no GitHub Actions?
O evento `pull_request_target` roda no contexto privilegiado do repositório base com acesso total a segredos de produção e tokens de escrita. Quando combinado com o checkout do código do fork enviado no pull request, ele permite que atacantes externos executem scripts arbitrários no runner e roubem credenciais corporativas.

### 2. Como a fixação de actions por commit SHA de 40 dígitos previne ataques à cadeia de suprimentos?
Tags como `@v3` ou `@main` são referências mutáveis no Git que podem ser alteradas se a conta do mantenedor for comprometida. O hash de commit SHA de 40 caracteres é criptograficamente imutável, garantindo que apenas a versão exata e auditada do código seja executada na esteira.

### 3. Qual o risco operacional de omitir o bloco de permissões no manifesto do workflow?
Na ausência de um bloco explícito de permissões, o token padrão herdado pelo runner assume os privilégios definidos nas configurações globais da organização, que frequentemente incluem permissão ampla de escrita. Uma dependência comprometida durante o build pode explorar esse token para adulterar releases ou modificar branches protegidas.

### 4. Como a filtragem de rede na saída (egress filtering) impede o vazamento de credenciais?
O bloqueio de saída restringe as conexões de rede do runner estritamente a domínios previamente homologados, como o próprio GitHub e repositórios oficiais de pacotes. Caso uma dependência adulterada tente abrir uma conexão reversa ou enviar variáveis de ambiente para a internet, a conexão é descartada imediatamente no nível do kernel.

### 5. O analisador estático em Python necessita de privilégios de administrador ou conexão com a internet?
Não. A ferramenta opera de forma totalmente offline e passiva, lendo os arquivos YAML locais diretamente do disco sem enviar telemetria externa e sem exigir privilégios de superusuário no sistema operacional.

***

## Conformidade Regulatória e Padrões Corporativos

Arquitetura defensiva e controles técnicos alinhados ao OWASP Top 10 CI/CD Security Risks, ao framework SLSA (Supply-chain Levels for Software Artifacts) e aos controles de integridade de software do CIS Benchmark.

***

## Links Úteis

* **Portal Corporativo:** https://ag.tavardt.com/br/
* **Suíte de Diagnósticos:** https://tools.tavardt.com/br/
* **Contato Técnico:** https://ag.tavardt.com/br/contato/

***

### Licença
Distribuído sob a Licença MIT. Consulte o arquivo [LICENSE](LICENSE) para obter detalhes completos.

***

© 2026 TAVARDT. Todos os direitos reservados.
