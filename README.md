# Entrega customizada Eduzz
---

A entrega customizada permite que você utilize qualquer serviço como plataforma de entrega de conteúdo na Eduzz, ou seja, quando um cliente comprar um produto, você receberá eventos em sua plataforma via requisições HTTP com os dados da venda, para que a liberação do conteúdo possa ser feita.

**Consideramos como sucesso todas as requisições que retornam o _[status HTTP 200](http://www.w3.org/Protocols/rfc2616/rfc2616-sec10.html)_**.

As configurações para o cadastro de entrega customizada podem serem acessadas no **[Órbita](https://orbita.eduzz.com/producer/webhook)**, na aba "Avançado" nas configurações do seu produto, caso você já tenha alguma entrega customizada, basta clicar na opção desejada na listagem, ou, caso ainda não possua uma entrega customizada, clique em "Adicionar Entrega", então, o seguinte modal onde você pode escolher o tipo de entrega e informar qual a url que deverá receber os dados será exibido:

![Modal de cadastro de entrega customizada no Órbita](https://github.com/eduzz/custom-delivery/raw/master/customizado_modal.png "Modal de cadastro de entrega customizada no Órbita")

## Autenticação

Para autenticar uma entrega customizada, recomendamos o uso do campo chave de origem, disponível também no serviço de **[Webhook](https://github.eduzz.com/eduzz/wrbhook)**.

A chave para integração pode ser visualizada em nossa plataforma na **[tela de configuração de chaves de api no Órbita](https://orbita.eduzz.com/producer/config-api)**.

Será enviado no payload da entrega customizada no campo edz_cli_origin_secret.

Ainda enviamos hoje o campo edz_cli_apikey por motivos de compatibilidade, porém, ele não deve mais ser utilizado e **será descontinuado em breve**.

### **origin_secret**

Para validar através do origin_secret basta comparar o valor recebido com o que será informado na **[tela de configuração de chaves de api no Órbita](https://orbita.eduzz.com/producer/config-api)**.

Segue exemplo abaixo:

```js
const { type, fields } = {...PAYLOAD_RECEBIDO_DA_EDUZZ};

// Sua chave de autenticação de requisições fornecida no Órbita (origin)
// https://orbita.eduzz.com/producer/config-api
const ORIGIN_SECRET = 'chave-de-autenticacao-fornecida-no-orbita';

const allow = type === 'create' && fields.edz_cli_origin_secret === ORIGIN_SECRET; // Deve permitir o acesso a plataforma
```

Dessa forma o usuário pode validar se as chaves são iguais, a origin_secret da conta do cliente é armazenada criptografada em nossos servidores então somente o cliente e as aplicações da Eduzz terão acesso a essa informação de forma programática.

### **sid**, **nsid**

Para autenticação da entrega podem serem utilizados também os campos de autenticação dos dados de envio, para gerar o **sid**, basta seguir os passos abaixo:

***Instale a biblioteca crypto:**

```sh
yarn add crypto
```

E então siga o seguinte snippet de código:

```js
const { type, sid, nsid, fields } = {...PAYLOAD_RECEBIDO_DA_EDUZZ};

const ksort = (obj) => {
  let keys = Object.keys(obj).sort();
  let sortedObj = {};

  for (let i in keys) {
    sortedObj[keys[i]] = obj[keys[i]];
  }

  return sortedObj;
}

const hash = Object.keys(ksort(fields)).reduce((acc, key) => `${acc}${fields[key]}`}, '');

const generatedSid = crypto.createHash('md5')
  .update(hash + fields.edz_cli_apikey)
  .digest('hex');

const generatedNsid = crypto.createHash('sha1')
    .update(`${fields.edz_fat_cod}${fields.edz_cnt_cod}${fields.edz_cli_cod}`)
    .digest('hex');

const sidOk = generatedSid === sid; // sid recebido é valido
const nsidOk = generatedNsid === nsid; // nsid recebido é valido

const allow = type === 'create' && sidOk && nsidOk; // Deve permitir o acesso a plataforma
```

Utilizando os campos nsid e sid gerados você pode validar se a requisição é segura.

### **vendas com order bump (múltiplos itens)**
Em casos de vendas com mais de um item, será enviado um item com o parâmetro **edz_order_bump_item = false**, sendo esse item o principal produto da venda.
Os demais itens da fatura, estarão marcados com o parâmetro de order bump (**edz_order_bump_item = true**).
Nesse caso, em ambas entregas, será enviado o campo **edz_has_order_bump=true**.

## Parâmetros
---

Campo     | Descrição | Tipo
------------- | ------------- | -----------------
edz_fat_cod | Id da fatura que originou a entrega | int
edz_fat_status | Status da fatura que originou a entrega | int
edz_fat_dtcadastro | Data em que a fatura foi cadastrada na Eduzz | Date
edz_order_bump_item | Informa se o item que está sendo entregue é um order bump | boolean
edz_has_order_bump | Informa se a venda (fatura) possui itens de order bump | boolean
edz_gtr_dist | Id do afiliado caso houver | int
edz_cnt_cod | Id do produto que está sendo entregue | int
edz_cnt_paicod | Id do produto pai do produto que está sendo entregue. É nulo caso o produto não tenha um pai | int
edz_cnt_titulo | Nome do produto que está sendo entregue | string
edz_cli_cod | Id do cliente | int
edz_cli_cel | Número de celular do cliente | string
edz_cli_cidade | Nome da cidade do cliente | string
edz_cli_uf | Sigla do estado do cliente | string
edz_cli_taxnumber | CPF/CNPJ do cliente | string
edz_cli_rsocial | Nome completo do cliente | string
edz_cli_email | Email do cliente | string
edz_valorpago | Valor do Produto | float
edz_trk_1      | Eduzz Tracker | String
edz_gtr_param1 | UTM Source | string
edz_gtr_param2 | UTM Campaign | string
edz_gtr_param3 | UTM Medium | string
edz_gtr_param4 | UTM Content | string
edz_produtor_email | O E-mail do produtor (Seu usuário) | string
edz_produtor_cod | O Id do produtor (Seu usuário) | int
edz_con_cod | Codigo do contrato caso seja assinatura | int
edz_con_status | Status do contrato | string
edz_con_status_cod | ID do status do contrato  | int
**edz_cli_apikey** | **Sua apikey de produtor, caso você já tenha gerado a nova versão da sua apikey serão enviados apenas os 4 últimos dígitos, caso contrário, iremos enviar a apikey inteira. Este campo será descontinuado em breve e deve ser utilizado o edz_cli_origin_secret** | **string**
**edz_cli_origin_secret** | **Sua origin key de produtor, esse campo deve ser validado com o valor que é exibido no [Órbita](https://orbita.eduzz.com/producer/config-api)** | **string**
**type** | **Operação de entrega ou remoção, quando você deve entregar, será enviado 'create', caso seja remoção de acesso (reembolso, atraso do contrato, etc) será enviado 'remove'** | **string**


## Tabela de status de faturas

Campo: **trans_status**

ID  | Status | Descrição
--- | ------ | -----------
1 | Aberta | Fatura aberta, cliente gerou boleto, mas ainda não foi compensado
3 | Paga | Compra foi paga, o cliente já esta apto a receber o produto
4 | Cancelada | Fatura Cancelada pela Eduzz
6 | Aguardando Reembolso | Cliente solicitou reembolso, porem o mesmo ainda não foi efetuado
7 | Reembolsado | Cliente já foi reembolsado pela eduzz
9 | Duplicada | Cliente tentou comprar mais de uma vez o mesmo produto, a segunda fatura fica como duplicada e não é cobrada
10 | Expirada | A fatura que fica mais de 15 dias aberta é alterada para expirada
11 | Em Recuperacao | Fatura entrou para o processo de recuperação
15 | Aguardando Pagamento | Faturas de recorrência após o vencimento ficam com o status aguardando pagamento
18 | Agendada | Fatura agendada pela recorrência para cobrar futuramente proximo a data de vencimento.

## Tabela de status de contratos

Campos: **edz_con_status_cod,edz_con_status**

ID  | Status | Descrição
--- | ------ | -----------
1 | Em dia | Contrato em dia
2 | Aguardando pagamento | Contrato com fatura em aguardando pagamento
3 | Suspenso | Contrato suspenso
4 | Cancelado | Contrato cancelado
7 | Atrasado | Contrato atrasado
9 | Finalizado | Contrato terminou
10 | Trial | Contrato em período trial
11 | Inadimplente | Contrato Inadimplente

## Suporte

Precisa de uma informação que não está na lista acima? Entre em contato com nosso suporte (suporte@eduzz.com), setor comercial (comercial@eduzz.com) ou através dos nossos canais no **[Órbita](https://orbita.eduzz.com)**.
