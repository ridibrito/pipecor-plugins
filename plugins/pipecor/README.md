# Plugin PipeCor

O [PipeCor](https://pipecor.com) é o CRM e financeiro de corretoras de planos
de saúde, seguros e consórcios. Este plugin conecta o Claude à conta PipeCor da
corretora e traz os procedimentos que o conector sozinho não ensina, começando
pela conciliação bancária das comissões recebidas das seguradoras e operadoras.

## O que o plugin acessa

- **Um único servidor:** o conector MCP do PipeCor, em
  `https://mcp.pipecor.com/mcp`, autenticado por OAuth com o login da própria
  corretora. As ferramentas respeitam as permissões de quem autorizou, e o
  PipeCor confere a conta e as permissões a cada chamada.
- **Nenhum código local:** o plugin não roda scripts nem hooks, não instala
  pacotes e não envia dados a nenhum outro endereço. Skills e comandos são só
  instruções em texto.
- **Dados:** ficam no PipeCor. O plugin não guarda nada. Política de
  privacidade: https://pipecor.com/politicas. Documentação do conector:
  https://pipecor.com/documentacao/mcp.

## Instalação (Claude Code / Cowork)

```
/plugin marketplace add ridibrito/pipecor-plugins
/plugin install pipecor@pipecor
```

No primeiro uso, o Claude pede para autorizar o PipeCor no navegador. O acesso
segue as permissões da pessoa que autorizou.

## Conteúdo

| Tipo | Nome | Para quê |
| --- | --- | --- |
| Skill | `conciliacao-bancaria` | Baixa das comissões a partir do extrato até o saldo bater |
| Comando | `/conciliar` | Atalho para iniciar a conciliação |

## Como adicionar uma skill

1. Crie `skills/<nome-da-skill>/SKILL.md` com o frontmatter `name` e
   `description`. A descrição diz **quando** usar a skill; é por ela que o
   Claude decide carregá-la.
2. Escreva o procedimento como para uma pessoa nova na corretora:
   - passos em ordem;
   - quais ferramentas `pipecor_*` usar e com que campos;
   - o que conferir;
   - quando parar e perguntar.
3. Toda regra deve vir de um caso real. Cite o caso: relatório, cliente,
   período.
4. Regra de negócio (validação, permissão, cálculo) não vai na skill: vai no
   MCP e no banco, para valer com ou sem plugin.
5. Suba a versão em `.claude-plugin/plugin.json` (0.x.y → 0.x+1.0 para skill
   nova, 0.x.y+1 para ajuste) e registre a mudança no `CHANGELOG.md`.
6. Valide antes do PR:

   ```
   claude plugin validate plugins/pipecor
   claude plugin validate .claude-plugin/marketplace.json
   ```

Quem tem o plugin recebe a versão nova com `/plugin marketplace update pipecor`
ou na próxima sessão, se a atualização automática estiver ligada.
