# Login GitHub – Vale na Conta

O painel atual em `/admin.html` usa token pessoal e continua ativo até validação do novo login.

## 1. Implantar o serviço Cloudflare Worker

Este serviço é necessário porque o GitHub Pages não executa código de autenticação no servidor.

1. Crie ou acesse sua conta em https://dash.cloudflare.com.
2. Em seu computador, use Node.js recente e, na pasta `cloudflare`, execute `npx wrangler login`.
3. Execute `npx wrangler deploy`. Anote a URL fornecida, como `https://vale-na-conta-admin.<sua-conta>.workers.dev`.
4. O serviço responderá com erro de configuração até receber os segredos descritos abaixo.

## 2. Criar GitHub OAuth App

1. Acesse https://github.com/settings/developers, vá em **OAuth Apps > New OAuth App**.
2. Application name: Vale na Conta Admin.
3. Homepage URL: URL do seu Cloudflare Worker.
4. Authorization callback URL: `https://SEU-WORKER.workers.dev/auth/callback` (substituir pela URL real).
5. Anote o **Client ID** e gere um **Client secret**.
6. Atenção: o escopo GitHub OAuth `public_repo` solicitado pelo serviço concede, durante a autorização, acesso de escrita aos repositórios públicos permitidos pela conta, não apenas ao Vale na Conta. O serviço restringe suas operações ao repositório `gmachado0/vale-na-conta`, mas esse escopo é mais amplo que um fine-grained PAT. Use uma conta com proteção 2FA e revogue a autorização caso não precise mais dela.

## 3. Configurar os três segredos

No diretório `cloudflare`, execute:

```bash
npx wrangler secret put GITHUB_CLIENT_ID
npx wrangler secret put GITHUB_CLIENT_SECRET
npx wrangler secret put SESSION_KEY
```

- Os dois primeiros vêm do GitHub OAuth App.
- `SESSION_KEY` deve ser um valor aleatório com pelo menos 32 caracteres, gerado localmente por gerenciador de senhas ou gerador seguro. Não use uma senha conhecida.
- Não coloque esses valores em arquivos rastreados pelo Git ou em mensagens do ChatGPT.
- Se houver erro `workers.dev`, configure o subdomínio Workers na Cloudflare.

## 4. Testar

1. Abra `https://SEU-WORKER.workers.dev/admin`.
2. Clique em **Entrar com GitHub** e autorize.
3. Confirme que aparece `Conectado como @gmachado0`.
4. Cadastre um restaurante de teste, publique e verifique em https://gmachado0.github.io/vale-na-conta/.
5. Faça logout e confirme que a publicação passa a exigir novo login.
6. Depois de validar, **revogue o token Fine-grained antigo** em https://github.com/settings/personal-access-tokens. Até lá, mantenha-o guardado em segurança.

## Arquitetura e segurança

- `admin-oauth.html` é a interface administrativa que o Worker serve na **mesma origem** que a API.
- `worker.js` administra OAuth, verifica o login `gmachado0` e a permissão de escrita do GitHub, mantém token OAuth cifrado em cookie `Secure; HttpOnly; SameSite=Lax` com validade de 7 dias, verifica a origem em chamadas de escrita e atualiza apenas `catalogo-oficial.json`.
- A sessão não é salva em localStorage e nenhum segredo OAuth fica no HTML público.
- `catalogo-oficial.json` é lido pelo site público automaticamente.
- Rascunhos da interface OAuth ficam em memória até serem publicados; exporte backup antes de fechar a aba.
- O Worker não está implantado automaticamente: a implantação depende de uma conta Cloudflare e da criação do OAuth App.
