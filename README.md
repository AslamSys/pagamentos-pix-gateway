# 💸 Finanças — PIX

## 🔗 Navegação

**[🏠 AslamSys](https://github.com/AslamSys)** → **[📚 _system](https://github.com/AslamSys/_system)** → **financas-pix**

### Containers Relacionados (finanças)
- [financas-contas](https://github.com/AslamSys/financas-contas)

---

**Container:** `financas-pix`  
**Stack:** Node.js + API Bacen  
**PSPs:** BB, Inter, Asaas

---

## 📋 Propósito

Gateway de integração com PSPs (Payment Service Providers) para envio e recebimento de PIX. Webhooks para notificações em tempo real.

---

## 🎯 Features

- ✅ Envio de PIX (QR Code, chave, copia e cola)
- ✅ Consulta de recebimentos
- ✅ Webhooks para notificações instantâneas
- ✅ Multi-PSP (BB, Inter, Asaas)
- ✅ Validação de chaves PIX

---

## 🔌 NATS Topics

### Subscribe
```javascript
Topic: "financas.pix.send"
Payload: {
  "pix_key": "+5511999998888",
  "amount": 150.00,
  "description": "Pagamento via Mordomo"
}
```

### Publish
```javascript
Topic: "financas.pix.sent"
Payload: {
  "txid": "E123456789202511271530",
  "status": "success",
  "amount": 150.00
}

Topic: "financas.pix.received"
Payload: {
  "txid": "E987654321202511271600",
  "payer": "Maria Silva",
  "amount": 200.00
}
```

---

## 🚀 Docker Compose

```yaml
pix-gateway:
  build: ./pix-gateway
  environment:
    - NATS_URL=nats://mordomo-nats:4222
    - PSP=asaas  # ou bb, inter
    - ASAAS_API_KEY=${ASAAS_API_KEY}
    - WEBHOOK_URL=https://mordomo.exemplo.com/webhooks/pix
  ports:
    - "4000:4000"
  deploy:
    resources:
      limits:
        cpus: '0.4'
        memory: 384M
```

---

## 🧪 Código

```javascript
const { connect } = require('nats');
const axios = require('axios');

const nc = await connect({ servers: process.env.NATS_URL });
const sc = StringCodec();

// Subscribe to send PIX
const sub = nc.subscribe('financas.pix.send');
for await (const msg of sub) {
    const { pix_key, amount, description } = JSON.parse(sc.decode(msg.data));
    
    // API Asaas
    const response = await axios.post('https://www.asaas.com/api/v3/payments', {
        customer: await resolveCustomer(pix_key),
        billingType: 'PIX',
        value: amount,
        description
    }, {
        headers: { 'access_token': process.env.ASAAS_API_KEY }
    });
    
    // Publish confirmation
    nc.publish('financas.pix.sent', sc.encode(JSON.stringify({
        txid: response.data.id,
        status: 'success',
        amount
    })));
}

// Webhook receiver
app.post('/webhooks/pix', (req, res) => {
    const { event, payment } = req.body;
    
    if (event === 'PAYMENT_RECEIVED') {
        nc.publish('financas.pix.received', sc.encode(JSON.stringify({
            txid: payment.id,
            payer: payment.customer.name,
            amount: payment.value
        })));
    }
    
    res.sendStatus(200);
});
```

---

## 📡 PSPs Suportados

```yaml
Asaas:
  - API: asaas.com/api/v3
  - Taxa: 0.69% por transação
  - Limite: R$ 10.000/transação

Banco do Brasil:
  - API: developers.bb.com.br
  - Requer certificado digital
  - Grátis para PJ

Inter:
  - API: developers.bancointer.com.br
  - Webhook nativo
  - R$ 1.49 por PIX
```

---

## 🔄 Changelog

### v1.0.0
- ✅ Asaas integration
- ✅ PIX send/receive
- ✅ Webhooks

---

## 🔐 Vault Integration

A `ASAAS_API_KEY` (e demais chaves de PSP) **não ficam em env var de produção** — são obtidas do `mordomo-vault` mediante autorização por voz.

```yaml
Credenciais gerenciadas pelo vault:
  - asaas_api_key
  - bb_cert
  - inter_api_key

Auth mode: voice
Pessoa autorizada: owner
Confiança mínima: 0.95
Módulo consumidor: mordomo-financas-pix
```

**Fluxo:** O `mordomo-orchestrator` só despacha a ação de PIX após receber a API key do vault, confirmando que foi o dono da casa quem pediu com confiança ≥ 0.95.

Veja: [mordomo-vault](https://github.com/AslamSys/mordomo-vault)
