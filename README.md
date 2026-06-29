# discord-net-components-v2

**Uma Skill para Claude (e outros LLMs) que ensina tudo sobre Discord.Net Message Components V2 (ComponentBuilderV2) em C#.**

Transforme seu bot Discord em uma interface moderna e rica com Containers, Sections, Text Displays, Media Galleries, Separators e mais — tudo com o poder do Discord Components V2.

![GitHub stars](https://img.shields.io/github/stars/linux-ur/discord-net-components-v2?style=social)
![License](https://img.shields.io/github/license/linux-ur/discord-net-components-v2)
![Discord.Net](https://img.shields.io/badge/Discord.Net-3.20+-blue)

## O que é isso?

Esta skill foi construída a partir da documentação oficial do **Discord.Net** (guides de Components V2) + referência da API do Discord. Ela dá ao Claude (Claude.ai, Claude Code ou via API) todo o conhecimento necessário para:

- Gerar UIs complexas com Components V2
- Escrever código fluente com `ComponentBuilderV2`
- Lidar com interações (botões, selects, modals, etc.)
- Evitar os erros comuns da migração V1 → V2

## Estrutura do projeto

```
discord-net-components-v2/
├── SKILL.md                          ← Ponto de entrada (mental model, cheat sheet de nesting, quick start)
└── references/
    ├── component-types.md            ← Especificação completa de cada tipo de componente
    ├── builder-guide.md              ← API fluente + exemplo completo
    ├── interactions.md               ← Como capturar interações, FindComponentById, UpdateAsync
    └── troubleshooting.md            ← Flag ComponentsV2, breaking changes v3.18, erros comuns
```

`SKILL.md` é curta e direta — carrega tudo que o LLM precisa pra tomar decisões rápidas. Os arquivos de referência são carregados sob demanda pra detalhes exaustivos.

## Instalação

### Claude.ai / Apps
1. Baixe o repositório como ZIP
2. Vá em **Settings → Capabilities → Skills**
3. Faça upload do ZIP da pasta `discord-net-components-v2`

### Claude Code
```bash
cp -r discord-net-components-v2 ~/.claude/skills/
```

### API / Outros LLMs
Inclua o conteúdo da skill na configuração de tools/skills conforme a documentação do seu agente.

Depois de instalada, é só falar com o Claude sobre **Discord.Net Components V2** que ele vai usar automaticamente.

## Uso rápido

Basta mencionar algo como:
- "Cria um painel com container, section e botões usando Components V2"
- "Como faço um modal com text input e checkbox no Discord.Net?"
- "Corrige esse código de ComponentBuilderV2"

## Escopo e Observações

- Baseado na versão **3.20.1** do Discord.Net
- Alguns componentes novos da API (Label, File Upload, etc.) estão documentados como "suporte na library ainda não confirmado"
- Se algo divergir do seu pacote instalado, confie no IntelliSense e abra um PR

## Contributing

PRs são bem-vindos! Especialmente:
- Confirmação dos métodos fluentes (`WithSection`, `WithContainer`, etc.)
- Suporte a novos componentes assim que o Discord.Net implementar
- Mais exemplos reais




## My other project that is the same thing but for python
[discordpy-components-v2](https://github.com/linux-ur/discordpy-components-v2)
## License

MIT — veja o arquivo [LICENSE](LICENSE).
