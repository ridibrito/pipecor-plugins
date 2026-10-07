---
name: conciliacao-bancaria
description: Conciliar o extrato bancário de uma corretora com o PipeCor — dar baixa nas comissões recebidas das seguradoras e operadoras, na data e no valor do banco, até o saldo do PipeCor bater com o extrato. Use quando o usuário pedir para conciliar, lançar recebimentos de comissão, dar baixa a partir de um extrato (PDF, CSV, OFX ou planilha) ou acertar o saldo/caixa de um período.
---

# Conciliação bancária no PipeCor

O objetivo é que o saldo diário de cada conta no PipeCor bata com o extrato do
banco. **O banco é a referência**: data e valor vêm do extrato, nunca da
previsão do PipeCor.

Este procedimento nasceu da conciliação de 18 meses de histórico de uma
corretora. Cada regra abaixo corrige um erro que aconteceu de verdade.

## Antes de começar

1. Confirme com o usuário a **conta** (ex.: Banco Principal, C6) e o
   **período**. Peça o extrato do banco e, se houver, os extratos ou
   demonstrativos das seguradoras (Tokio, Zurich, Icatu, Amil etc.). São eles
   que dizem qual cliente e qual parcela cada crédito paga.
2. Pergunte o **saldo inicial** da conta e a data dele, e confira na tela
   *Lançamentos › Contas e conciliação*. Saldo inicial de R$ 0,00 com data é
   um saldo informado.
3. Monte uma lista dos créditos do extrato (data, valor, histórico). Trabalhe
   crédito a crédito e mantenha essa lista atualizada com o status de cada um.

## Para cada crédito de comissão

### 1. Achar a parcela

- `pipecor_search_commissions` com `status` (`forecast` ou `approved`) e um
  intervalo `from`/`to` em torno do vencimento. O `id` devolvido é o que a
  baixa espera.
- Ou `pipecor_search_receivables`: cada conta traz `commissions` (id e status).
  O id da conta também é aceito pela baixa quando ela tem **uma única**
  comissão aberta; com mais de uma, use o id da comissão.
- Identifique pela combinação de seguradora, cliente, número da parcela
  (`installment_number`/`total_installments`) e valor. **Nunca escolha no
  chute**: se dois candidatos servem, pergunte ao usuário.

### 2. Dar a baixa

`pipecor_receive_commission` com:

| Campo | O que enviar |
| --- | --- |
| `id` | id da comissão |
| `amount` | valor **creditado no banco** para esta parcela |
| `payment_date` | data do crédito no extrato (AAAA-MM-DD) |
| `final` | `false` se ainda vem outro crédito desta parcela; senão omita |
| `notes` | origem: seguradora, nº do extrato, cliente, bruto/ISS/IR |
| `idempotency_key` | única por crédito (ver abaixo) |

Casos:

- **Parcela paga em mais de um crédito** (ex.: R$ 142,90 = 119,08 + 23,82):
  uma chamada por crédito, cada uma na sua data. Todas com `final: false`,
  menos a última. A parcela fecha sozinha quando a soma alcança o previsto.
  Nunca some os créditos numa baixa só: o caixa ficaria errado nos meses entre
  eles.
- **Um crédito que paga várias parcelas** (ex.: "RECEBIMENTOS AMIL, lote com 5
  parcelas"): uma chamada por parcela, todas na data do crédito, com o lote nas
  `notes`. A soma das chamadas tem que dar o valor do crédito.
- **Valor acima do previsto**: é aceito. A diferença volta em
  `receipt.adjustment`. Conte ao usuário, porque costuma indicar previsão
  errada na tabela de comissão.
- **Valor abaixo do previsto**: descubra o motivo **antes** da baixa.
  - Vem outro crédito depois → `final: false`.
  - É imposto retido na fonte (ISS, IR) → baixa final pelo líquido, com bruto,
    ISS e IR nas `notes`. Avise que a diferença fica como baixa contábil.
  - Não se sabe → pergunte. Não feche a parcela por conta própria.

### 3. Conferir a resposta

- `receipt.open_after` diz quanto falta na parcela; `receipt.final` diz se ela
  fechou.
- `receivable.status = settled` significa que a conta a receber foi baixada e
  o dinheiro entrou no caixa. Com `partial: true`, ela segue aberta para o
  próximo crédito.
- Marque o crédito como conciliado na sua lista.

## Chave de idempotência

- Uma chave nova **por crédito**. Formato sugerido:
  `conc-AAAAMMDD-<8 primeiros do id da comissão>-<n>`, com `n` contando os
  créditos daquela parcela.
- Reenvie com a **mesma chave e os mesmos argumentos** só para repetir uma
  chamada cujo resultado você não viu (timeout, erro de rede).
- `receivable_settlement_failed`: a comissão foi baixada e a conta a receber
  não. Repita com a mesma chave e os mesmos argumentos para concluir.
- `idempotency_conflict`: a chave já foi usada com outros dados. Não troque os
  argumentos para "forçar". Descubra o que foi gravado antes.

## Erros e o que fazer

| Erro | Significa | Ação |
| --- | --- | --- |
| `not_found_or_not_allowed` | id errado | buscar o id em `pipecor_search_commissions` |
| `ambiguous_receivable` | conta com várias comissões abertas | escolher uma das listadas em `commissions` |
| `invalid_commission_state` | parcela já quitada ou estornada | ler `details.commission` e mostrar ao usuário (ver "Limites") |
| `invalid_amount` | zero, negativo ou mais de 2 casas | corrigir o valor |
| `invalid_arguments` | campo a mais ou tipo errado | a mensagem diz qual campo |

## Créditos que não são comissão

- **Bonificação de seguradora, aporte, reembolso, transferência entre
  contas**: não use `pipecor_receive_commission`.
- **Bonificação** não deve ser lançada por *Financeiro › Bonificações* →
  "Marcar recebida": hoje isso **não lança no caixa**. Peça ao usuário um
  lançamento avulso de receita em *Lançamentos* e não cadastre a mesma
  bonificação no módulo, para não duplicar.
- **Débitos do extrato**: confira com `pipecor_search_payables` e
  `pipecor_search_payouts`. Repasse pago ao corretor se registra com
  `pipecor_pay_payout` ou pelo lote.

## Fechamento

1. Some, por dia, as entradas e saídas conciliadas e compare com o saldo do
   extrato no fim de cada dia. Se o conector expõe operações do dono, use
   `financial.getCashFlow` via `pipecor_describe_operation` e
   `pipecor_run_read_operation` (só leitura). Senão, peça ao usuário o saldo
   da tela *Contas e conciliação* no último dia.
2. Entregue um resumo com:
   - créditos conciliados (data, valor, parcela);
   - créditos sem par no PipeCor;
   - parcelas com crédito parcial ainda aberto;
   - ajustes acima do previsto;
   - diferenças lançadas como baixa contábil;
   - saldo final do PipeCor × saldo do banco.

## Limites atuais do PipeCor (contornos)

- **Baixa com data, valor ou conta errada** não se edita pelo conector.
  Peça ao usuário para usar *Comissões › ⋮ › Desfazer confirmação* e depois
  refaça a baixa pelo conector.
- **Saldo inicial**: o aviso "nenhuma conta tem saldo inicial" aparece mesmo
  com saldo de R$ 0,00 informado. Pode ignorar se a data estiver preenchida.
- **Venda na linha "Outros"**: não tem fornecedor. Não envie
  `supplier_provider_id`; se for importante, ponha o nome em `product_name`.
