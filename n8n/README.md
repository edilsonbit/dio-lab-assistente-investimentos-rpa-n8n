# Workflow N8N — Assistente de Investimentos

`workflow.json` reúne o **MVP (mensagens fixas)** e o **desafio (Agente de IA)** em um só fluxo:

```
Webhook → Baixar Investimentos → Cruzar Dados → Usar IA?
                                                 ├─ sim → Agente de IA (OpenAI) → Formatar Resposta IA ─┐
                                                 └─ não → Mensagem Estática ────────────────────────────┴→ Email Válido?
                                                                                                             ├─ sim → Enviar Email (Gmail) ───────┐
                                                                                                             └─ não → Registrar Email Inválido ───┴→ Responder Webhook
```

## Como importar

1. No N8N: **Workflows → Import from File** e escolha `workflow.json`.
2. No nó **OpenAI Chat Model**, crie a credencial da OpenAI. No n8n Cloud, a tela de credencial oferece **créditos gratuitos da OpenAI** (cota limitada de chamadas) — use essa opção; fora dela, é preciso uma chave da [plataforma OpenAI](https://platform.openai.com/api-keys), que é cobrada à parte do ChatGPT.
3. No nó **Enviar Email (Gmail)**, conecte sua conta Google (OAuth, sem custo) e troque `SEU_EMAIL_DE_TESTE@gmail.com` pelo seu e-mail de teste.
4. Ative o workflow e use no notebook a URL de produção: `https://<sua-instancia>.app.n8n.cloud/webhook/clientes`.
   A URL `webhook-test/` só responde enquanto o editor está em "Listen for test event".

## Escolhendo o modo

| Chamada | Resultado |
|---|---|
| `POST .../webhook/clientes` | Mensagens geradas pelo Agente de IA (padrão) |
| `POST .../webhook/clientes?modo=template` (ou `"modo": "template"` no JSON) | Mensagens fixas por perfil (MVP) |

O webhook só responde depois do envio dos e-mails; para os enviados, a resposta traz o retorno do Gmail (`id`, `threadId`, `labelIds`), e para os inválidos, o cliente com `status: não enviado: e-mail inválido`.

## Decisões técnicas

- **Um fluxo, dois modos:** o cruzamento de dados é o mesmo nos dois casos; o nó `Usar IA?` só decide quem escreve a mensagem. A IA é o padrão; o modo `template` mantém o MVP funcionando sem gastar créditos de IA.
- **CSV lido uma vez:** `Baixar Investimentos` usa *Execute Once* e o nó `Cruzar Dados` processa todos os clientes juntos, em vez de baixar o CSV por cliente.
- **Produtos que cabem no saldo:** além do perfil, o cruzamento filtra pelo aporte `minimo` do CSV. Se o saldo não cobrir nenhum, sugere o produto de menor mínimo do perfil e a mensagem avisa.
- **IA com limites:** o prompt de sistema obriga o modelo a citar só produtos da lista recebida e a não prometer rentabilidade, para não inventar dados financeiros.
- **E-mail só para o endereço de teste:** os e-mails dos clientes são fictícios, mas `email.com` é um domínio real. Por isso todas as mensagens vão para um único e-mail de teste, e a mensagem já cita o cliente pelo nome. Nunca use `{{ $json.email }}` no campo **To**: o envio iria de verdade para os endereços da tabela.
- **Assunto gerado junto com a mensagem:** a IA responde um JSON `{assunto, mensagem}`; se vier fora do formato, o texto vira a mensagem e o assunto cai no padrão `Sugestão de Investimento: <produto>`, o mesmo usado no modo `template`.
- **E-mail inválido não quebra o fluxo:** o nó `Email Válido?` testa o formato do e-mail do cliente; os inválidos não vão para o Gmail e seguem para `Registrar Email Inválido`, que marca `status: não enviado: e-mail inválido`. Para testar, altere um e-mail no payload do notebook (ex.: `ana@`).
- **OpenAI `gpt-4o-mini`:** modelo barato e suficiente para textos curtos, e coberto pelos créditos gratuitos do n8n Cloud. Para trocar de provedor, basta substituir o nó de modelo ligado ao Agente.
