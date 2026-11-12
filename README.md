# Servidor de WhatsApp — NexPlay

Este pacote conecta o seu número de WhatsApp ao painel NexPlay por QR Code.
Ele recebe as mensagens, envia para o robô de IA do painel e devolve a resposta ao cliente.

## O que tem no pacote

| Arquivo | Para que serve |
|---|---|
| `server.js` | o servidor (QR Code, status, envio e recebimento) |
| `package.json` | o que o servidor precisa para funcionar |
| `Dockerfile` | receita de instalação usada pelo Render |
| `render.yaml` | configura o Render sozinho (plano Starter + disco + relógio dos testes) |
| `.dockerignore` / `.gitignore` | arquivos que não devem subir |

Todos os arquivos ficam na **raiz** do repositório — não crie subpasta.

---

## Parte 1 — Subir no GitHub

1. Acesse github.com e clique em **New repository**.
2. Dê um nome, por exemplo `nexplay-whatsapp`, marque **Private** e clique em **Create repository**.
3. Clique em **Add file → Upload files**.
4. Arraste **todos** os arquivos deste pacote.
5. Confirme que `package.json`, `Dockerfile` e `render.yaml` aparecem na tela principal do repositório.
6. Clique em **Commit changes**.

> Os arquivos que começam com ponto (`.dockerignore` e `.gitignore`) podem ser criados pelo botão
> **Add file → Create new file**, digitando o nome exato.

---

## Parte 2 — Subir no Render

1. Acesse render.com e entre com a conta do GitHub.
2. Clique em **New → Blueprint**.
3. Escolha o repositório criado.
4. Confira apenas dois pontos:
   - **Plano: Starter** (o plano Free dorme e não tem disco, então a sessão do WhatsApp se perde).
   - **Disco montado em `/var/data`**.
5. Clique em **Apply** e espere o deploy ficar **Live** (3 a 8 minutos).
6. Copie o endereço mostrado no topo, algo como `https://nexplay-whatsapp.onrender.com`.

---

## Parte 3 — Testar antes de conectar

Abra no navegador:

```
https://SEU-ENDERECO.onrender.com/api/diagnostico
```

O ideal é ver:

- `"ok": true`
- `"pasta_persistente": true`
- `"permissoes_banco": "completa"`

Se aparecer algo na lista `problemas`, leia a mensagem — quase sempre é o disco fora de `/var/data`.

---

## Parte 4 — Conectar no painel

1. Abra a aba **WhatsApp** no painel.
2. Cole o endereço do servidor e clique em **Salvar**.
3. Clique em **Testar servidor**.
4. Clique em **Gerar QR Code** e leia no celular em *WhatsApp → Dispositivos conectados → Conectar dispositivo*.
5. Quando conectar, a tela mostra o número e o nome do perfil.

---

## Atualizar um servidor que já está no ar

O servidor precisa estar na versão **1.7.0**. Essa versão reúne tudo:

- responde os contatos novos do WhatsApp (que chegam com identificador interno, o "LID");
- confirma no painel se a resposta da IA saiu;
- **repassa as fotos enviadas pelos clientes** — é a foto da tela do aplicativo que libera o teste gratuito;
- **reconhece as mensagens que o dono envia pelo celular**, pausando a IA naquela conversa;
- **liga o relógio dos testes**: a cada 5 minutos o servidor avisa o Backend para cortar os testes
  gratuitos que já venceram, mesmo com o site parado e sem ninguém usando (veja a seção abaixo).

1. Baixe o pacote de novo pelo botão **Baixar servidor (.zip)** na aba WhatsApp.
2. No GitHub, abra o repositório do servidor → **Add file → Upload files**.
3. Envie `server.js` (os outros arquivos não mudaram) e confirme em **Commit changes**.
4. O Render faz o deploy sozinho, aguarde ficar **Live**.
5. Volte ao painel e confirme em `/api/diagnostico` que aparece `"versao": "1.7.0"` e, dentro de
   `relogio_testes`, `"ligado": true`.
6. Depois disso, mande uma mensagem de outro celular para o número conectado: o robô deve responder em poucos segundos.
7. Envie uma foto qualquer pelo WhatsApp: o robô deve responder confirmando o recebimento. Se responder isso, o teste automático está funcionando.
8. Responda você mesmo pelo celular para um cliente: a IA deve ficar calada naquela conversa por 1 hora — sem responder às suas mensagens.

---

## Rotas disponíveis

| Rota | Método | Função |
|---|---|---|
| `/api/diagnostico` | GET | verifica se está tudo certo |
| `/api/status` | GET | conexão, número, nome e QR |
| `/api/conectar` | POST | inicia a conexão e devolve o QR Code |
| `/api/qr` | GET | devolve o QR Code atual |
| `/api/desconectar` | POST | desconecta e apaga a sessão |
| `/api/enviar` | POST | envia mensagem (`numero` e `mensagem`) |
| `/api/agente` | POST | repassa uma consulta ao robô do painel |
| `/api/guardiao` | GET | situação do relógio dos testes (última e próxima execução) |
| `/api/guardiao` | POST | força uma verificação dos testes agora, sem esperar os 5 minutos |

---

## Relógio dos testes gratuitos (a cada 5 minutos)

O painel do provedor trabalha com **data inteira**, então a linha de um teste de 6 horas ficaria
liberada lá até o fim do dia — e o cliente continuaria com acesso em aplicativos externos depois
das 6 horas. O Backend tem uma rotina de corte, mas antes ela só era acionada quando alguém abria
o player, o painel ou o link público do teste.

Agora este servidor faz o papel do relógio: enquanto ele estiver no ar (plano Starter, sem dormir),
chama a rotina de corte a cada 5 minutos. Se o servidor ficar fora do ar ou dormir, o relógio para
junto — por isso o plano precisa ser Starter.

Como conferir:

1. Abra `https://SEU-ENDERECO.onrender.com/api/guardiao`.
2. Você deve ver `"ligado": true`, `"intervalo_minutos": 5` e uma `"ultima_execucao"` de poucos minutos atrás.
3. `"cortados_total"` é o quanto já foi cortado desde que o servidor subiu.

Se `"ultimo_erro"` aparecer, o mais comum é o Backend não responder ou a variável `GUARDIAO_KEY`
não estar configurada no Render. Fora isso, não é preciso mexer em nada: o relógio roda sozinho.

Quer testar na hora? Abra o painel, gere um teste e espere passar o prazo. Ou facilite a conferência
abrindo `https://SEU-ENDERECO.onrender.com/api/guardiao` e procurando a `proxima_execucao`.

---

## Avisos importantes

- Depois do primeiro QR Code lido, a sessão fica salva no disco e o servidor reconecta sozinho.
- Não troque o plano para Free: ele dorme e perde a sessão.
- Mantenha o endereço do serviço em sigilo — é o controle do seu WhatsApp.
- A IA só responde se a chave do Google Gemini (GEMINI_API_KEY) estiver cadastrada no Backend. Sem ela, as mensagens chegam e ficam aguardando você.