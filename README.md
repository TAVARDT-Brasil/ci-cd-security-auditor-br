# ci-cd-security-auditor-br

Auditor estatico e validador de postura defensiva para workflows do GitHub Actions contra roubo de secrets de CI/CD, injeção de scripts e supply chain poisoning.

## Acesso Direto
Acesse a versao interativa oficial hospedada no GitHub Pages:
https://tavardt-brasil.github.io/ci-cd-security-auditor-br/

## Vulnerabilidades Inspecionadas
1. **pull_request_target Inseguro:** Vetor critico responsavel por vazamentos em massa de credenciais atraves de forks nao confiaveis.
2. **Permissoes Excessivas no GITHUB_TOKEN:** Deteccao de `permissions: write-all` e escopos globais nao mitigados.
3. **Supply Chain Risks em Actions:** Identificacao de dependencias sem fixacao por commit SHA imutavel.
4. **Command Injection em Blocos run:** Deteccao de variaveis de contexto interpoladas diretamente em shell script.

## Licenca
Licenciado sob a Apache License 2.0. Desenvolvido pela divisao de Ciberseguranca e Engenharia Digital da TAVARDT.
