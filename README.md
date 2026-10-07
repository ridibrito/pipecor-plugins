# Plugins do PipeCor para o Claude

Marketplace público dos plugins do [PipeCor](https://app.pipecor.com) para
Claude Code e Cowork.

## Instalar

```
/plugin marketplace add ridibrito/pipecor-plugins
/plugin install pipecor@pipecor
```

No primeiro uso, o Claude pede para autorizar o PipeCor no navegador. O plugin
não guarda senha nem token, e cada corretora acessa só os próprios dados, com
as permissões de quem autorizou.

## Plugins

| Plugin | O que traz |
| --- | --- |
| [`pipecor`](plugins/pipecor) | Conector MCP do PipeCor, skill de conciliação bancária e comando `/conciliar` |

## Atualizações

Novas skills entram por PR neste repositório, com a versão do plugin
aumentada. Para receber a versão nova:

```
/plugin marketplace update pipecor
```

Veja o [CHANGELOG](plugins/pipecor/CHANGELOG.md).
