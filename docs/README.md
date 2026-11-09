# Documentação do NɅCO

Este diretório reúne a documentação funcional, técnica e de planejamento do NɅCO. A especificação de imagens Q63–Q103 é complementar e independente do registro QU do RNF-01.

## Requisitos e escopo

- [Especificação consolidada](requirements/01_ESPECIFICACAO_CONSOLIDADA.md)
- [Decisões complementares do MVP](requirements/NACO_DECISOES_COMPLEMENTARES_MVP.md)

## Arquitetura

- [Modelo de dados e API](architecture/02_MODELO_DADOS_E_API.md)
- [Armazenamento e processamento de imagens (Q63–Q103)](architecture/05_ARMAZENAMENTO_E_PROCESSAMENTO_DE_IMAGENS.md)
- [Armazenamento e processamento de imagens (PDF)](architecture/05_ARMAZENAMENTO_E_PROCESSAMENTO_DE_IMAGENS.pdf)
- [Arquitetura e especificação técnica](architecture/NACO_ARQUITETURA_ESPECIFICACAO_TECNICA.md)
- [Arquitetura e especificação técnica (PDF)](architecture/NACO_ARQUITETURA_ESPECIFICACAO_TECNICA.pdf)

## Decisões

- [Registro de decisões e RNF-01](decisions/03_REGISTRO_DECISOES_RNF01.md)

## Planejamento

- [Plano de desenvolvimento e cronograma](planning/NACO_PLANO_DESENVOLVIMENTO_CRONOGRAMA.md)
- [Plano de desenvolvimento e cronograma (PDF)](planning/NACO_PLANO_DESENVOLVIMENTO_CRONOGRAMA.pdf)

## Testes e auditoria

- [Relatório comparativo e auditoria](testing/04_RELATORIO_COMPARATIVO_E_AUDITORIA.md)

## Contrato da API

- O arquivo `api/openapi.yaml` ainda não foi criado; sua produção está planejada para a etapa de desenvolvimento, a partir das rotas, DTOs e convenções já consolidados em `architecture/02_MODELO_DADOS_E_API.md` e `requirements/NACO_DECISOES_COMPLEMENTARES_MVP.md`.
- Cloudinary foi escolhido para o MVP via `StoragePort`; o contrato HTTP do upload pelo responsável e o mecanismo operacional de reprocessamento de exclusão serão definidos antes de implementar essas operações.
- A documentação lógica não representa schema Prisma, migrations, código ou testes já executados.

## Recursos visuais

A pasta `assets/` está reservada para imagens e diagramas da documentação. 