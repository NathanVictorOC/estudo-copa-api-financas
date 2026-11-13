# Estudo — Copa da API Finanças

**Disciplina:** Desenvolvimento Web Back-end
**Aluno:** _Nathan Victor de Oliveira Costa_
**Equipe no Projeto 2:** _João Pedro Aleksandrov Lorenzetti e Eduardo de Oliveira Aguiar_
**Data:** 13/11/2026

Registro da atividade de estudo do roteiro da Copa da API Finanças (Projeto 2 — API de Controle Financeiro). Abaixo estão a resolução dos 6 treinos e o raciocínio de cada um.

---

## Treino 1 — Códigos de status

| # | Situação | Status | Por quê |
|---|----------|--------|---------|
| 1 | `POST /categorias` com nome que o usuário já tem para o mesmo tipo | **409** | Nome repetido é conflito (Conflict). |
| 2 | `PUT /fontes-destinos/12` com `tipo: "poupanca"` | **400** | Valor de `tipo` fora dos permitidos é dado inválido (Bad Request). *Conferir na seção 6 da descrição do projeto se "poupanca" não é um tipo válido.* |
| 3 | `GET /relatorios/categorias` sem `inicio` e `fim` | **400** | Parâmetros obrigatórios ausentes. |
| 4 | `POST /auth/login` com a senha errada | **401** | Credenciais erradas no login retornam 401. |
| 5 | `GET /rota-que-nao-existe` | **404** | Rota inexistente. |
| 6 | `POST /lancamentos` válido | **201** | Criação de lançamento bem-sucedida. |

---

## Treino 2 — Caça ao bug

Trecho original:

```javascript
// routes/fontesDestinos.js
router.delete('/:id', async (req, res) => {
  const [r] = await db.query(
    `DELETE FROM fontes_destinos WHERE id = ${req.params.id} AND usuario_id = ${req.auth.id}`
  );
  if (r.affectedRows === 0) {
    res.status(404).json({ erro: 'Fonte/destino não encontrada.' });
  }
  res.status(204).end();
});
```

**Erro principal:** o SQL é montado com *template string* (`${req.params.id}`), em vez de consulta parametrizada com `?`. Isso permite **SQL injection**.

**Regra violada:** consultas sempre parametrizadas com `?`.

**Outros dois problemas do trecho:**

1. **Falta `return` depois da resposta de erro.** Quando não encontra o registro, responde 404 e continua para `res.status(204).end()`, gerando duas respostas para a mesma requisição (erro `ERR_HTTP_HEADERS_SENT`).
2. **SQL dentro da rota.** O SQL deve ficar somente em `models/`; a rota só chama a função do model.

(O trecho acerta ao filtrar por `usuario_id` vindo de `req.auth.id`, o que garante o isolamento entre usuários, e ao devolver 404 se o registro não existe ou é de outro usuário.)

**Correção:**

```javascript
// models/fonteDestino.js
async function excluir(id, usuarioId) {
  const [r] = await db.query(
    'DELETE FROM fontes_destinos WHERE id = ? AND usuario_id = ?',
    [id, usuarioId]
  );
  return r.affectedRows > 0;
}

// routes/fontesDestinos.js
router.delete('/:id', async (req, res) => {
  try {
    const removido = await FonteDestino.excluir(req.params.id, req.auth.id);
    if (!removido) {
      return res.status(404).json({ erro: 'Fonte/destino não encontrada.' });
    }
    return res.status(204).end();
  } catch (err) {
    console.error(err); // detalhe só no log do servidor
    return res.status(500).json({ erro: 'Erro interno.' });
  }
});
```

Observação: se a fonte/destino tiver lançamentos vinculados, o banco recusa a exclusão por chave estrangeira, e a rota deve responder **409** (exclusão com vínculos).

---

## Treino 3 — Parcelas em centavos

Compra de **R$ 250,00 em 7×**.

- `totalCentavos = 25000`
- `base = floor(25000 / 7) = 3571` centavos
- `resto = 25000 − 3571 × 7 = 25000 − 24997 = 3` centavos (vai para a 1ª parcela)

| Parcela | Valor |
|---------|-------|
| 1/7 | R$ 35,74 |
| 2/7 | R$ 35,71 |
| 3/7 | R$ 35,71 |
| 4/7 | R$ 35,71 |
| 5/7 | R$ 35,71 |
| 6/7 | R$ 35,71 |
| 7/7 | R$ 35,71 |

**Conferência:** 35,74 + 6 × 35,71 = 35,74 + 214,26 = **R$ 250,00** ✔

---

## Treino 4 — Fatura de cada parcela

Cartão com **fechamento dia 18** e **vencimento dia 3**; compra de **R$ 250,00 em 2×** em **18/08/2026**.

Raciocínio:

- Valor das parcelas: 25000 / 2 = 12500 centavos, resto 0 → **R$ 125,00** cada.
- A compra foi **no dia** do fechamento (18), então a 1ª parcela entra na fatura do **mês seguinte**: fechamento em **18/09/2026**.
- Vencimento (3) é **menor** que o fechamento (18), então vence no **mês seguinte** ao fechamento: **03/10/2026**.
- Referência = mês do vencimento: **2026-10**.
- A 2ª parcela cai 1 mês depois.

| Parcela | Valor | Fechamento | Vencimento | Referência |
|---------|-------|------------|------------|------------|
| 1/2 | R$ 125,00 | 18/09/2026 | 03/10/2026 | 2026-10 |
| 2/2 | R$ 125,00 | 18/10/2026 | 03/11/2026 | 2026-11 |

---

## Treino 5 — Fatura com fevereiro

Cartão com **fechamento dia 31** e **vencimento dia 8**; compra de **R$ 120,00 em 2×** em **31/01/2026**.

Raciocínio:

- Valor das parcelas: 12000 / 2 = 6000 centavos → **R$ 60,00** cada.
- A compra foi **no dia** do fechamento (31), então a 1ª parcela entra na fatura do **mês seguinte**, ou seja, fevereiro.
- Fevereiro de 2026 tem 28 dias (2026 não é bissexto). O dia 31 não existe, então vira o **último dia do mês**: fechamento em **28/02/2026**.
- Vencimento (8) é **menor** que o fechamento (31), então vence no mês seguinte ao fechamento: **08/03/2026**. Referência **2026-03**.
- A 2ª parcela cai 1 mês depois: fechamento em março (dia 31 existe) em **31/03/2026**, vencimento **08/04/2026**, referência **2026-04**.

| Parcela | Valor | Fechamento | Vencimento | Referência |
|---------|-------|------------|------------|------------|
| 1/2 | R$ 60,00 | 28/02/2026 | 08/03/2026 | 2026-03 |
| 2/2 | R$ 60,00 | 31/03/2026 | 08/04/2026 | 2026-04 |

---

## Treino 6 — Por que `UNIQUE` no banco?

Mesmo que a rota faça um `SELECT` antes do `INSERT`, duas requisições simultâneas com o mesmo e-mail podem passar pelo `SELECT` ao mesmo tempo (ambas veem que o e-mail não existe) e depois as duas inserem, criando duplicidade. O `SELECT` e o `INSERT` são dois passos separados, então há uma janela de concorrência (*race condition*). O `UNIQUE` faz o próprio banco garantir a regra de forma atômica: a segunda inserção falha com `ER_DUP_ENTRY`, e a rota trata esse erro respondendo **409**. O `SELECT` pode continuar existindo para dar uma mensagem amigável, mas a garantia final é do banco.

---
