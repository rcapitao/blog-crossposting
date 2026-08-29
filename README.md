# blog-crossposting

Automação que crossposta novos posts do blog [rcapitao.com](https://rcapitao.com) (gerado com [Hugo](https://gohugo.io/), fonte em [rcapitao/rcapitao-vhugo](https://github.com/rcapitao/rcapitao-vhugo)) para o **Mastodon** e o **Bluesky**, lendo o feed RSS do blog.

## Visão geral

A automação é orientada a evento, com um polling esporádico como rede de segurança:

1. Quando um post novo é publicado, o workflow de deploy do blog ([`rcapitao/rcapitao-vhugo`](https://github.com/rcapitao/rcapitao-vhugo), `.github/workflows/hugo.yml`) termina de publicar o site no GitHub Pages e, no último step, dispara um evento `repository_dispatch` (`blog-published`) para este repositório.
2. Isso aciona imediatamente o workflow `.github/workflows/crosspost.yml` aqui, que instala as dependências Python e executa `crosspost.py`.
3. Como rede de segurança, o mesmo workflow também roda por `schedule` a cada 3h (08h-23h horário de Brasília), caso o disparo do passo 1 falhe por algum motivo (token expirado, erro de rede, etc.). Também pode ser disparado manualmente.
4. `crosspost.py` lê o feed RSS configurado em `FEED_URL`.
5. Compara os links dos posts do feed com os já registrados em `state.json` (o "registro do que já foi publicado").
6. Para cada post novo (do mais antigo para o mais recente), publica uma mensagem no Mastodon e no Bluesky.
7. Atualiza `state.json` com os links recém-publicados e o workflow faz commit + push automático desse arquivo no repositório.

Não há servidor rodando 24/7 — a detecção é orientada a evento (`repository_dispatch` do repositório do blog), com **polling** esporádico do feed RSS como rede de segurança.

## Formato da mensagem publicada

```
Título do post: link

Meta description do post
```

- A primeira linha junta o título e o link do post, separados por `:`.
- Há uma linha em branco entre o título/link e a meta description.
- A meta description é extraída do campo `summary`/`description` do RSS (com tags HTML removidas).
- Se o post não tiver meta description, é usado o conteúdo completo do post (campo `content` do RSS, com tags HTML removidas) na segunda parte da mensagem.
- A mensagem é truncada (com reticências `…`) para caber no limite de cada rede: 480 caracteres para o Mastodon, 290 para o Bluesky. Isso é necessário porque o RSS do blog inclui o conteúdo completo do post na descrição, não um resumo curto.
- No Bluesky, o link na primeira linha é publicado como link clicável (rich text facet), não como texto simples.
- No Bluesky, o post também inclui um card de preview do link (imagem `og:image` da página do post, título e descrição), igual ao que aparece ao colar um link manualmente. Se a página não tiver `og:image`, o post é publicado sem o card.
- Se o post não tiver meta description nem conteúdo, só é publicada a primeira linha.

## Redes suportadas

| Rede | Suportado | Autenticação |
|---|---|---|
| Mastodon | ✅ | Access token de uma aplicação OAuth |
| Bluesky | ✅ | App Password (AT Protocol) |
| Threads | ❌ (por agora) | — |

Cada rede é independente: se as variáveis/secrets de uma rede não estiverem definidas, o script simplesmente ignora essa rede e publica só nas demais. Mais importante ainda: o progresso de publicação é rastreado **por rede** em `state.json` (não é "tudo ou nada"). Se a publicação no Mastodon tiver sucesso mas a do Bluesky falhar, a próxima execução tenta de novo **só o Bluesky** — o post não é republicado no Mastodon.

### Sobre o Threads

O Threads (Meta) não tem uma forma simples de publicar sem usar a API oficial do Threads, que exige criar um app no Meta for Developers, vincular uma conta Instagram Business e passar por processo de revisão. Por isso não está incluído por agora — pode ser adicionado depois se você quiser avançar com esse processo.

## Configuração

### 1. Descobrir a URL do feed RSS

O feed RSS deste blog está em `https://rcapitao.com/feed.xml`.

### 2. Criar o token do Mastodon

1. Entre na sua instância Mastodon (web).
2. Vá em **Preferências → Desenvolvimento → Nova aplicação**.
3. Dê um nome (ex: `blog-crossposting`) e marque o scope `write:statuses`.
4. Crie a aplicação e copie o **access token** gerado.
5. Anote também a URL base da sua instância (ex: `https://mastodon.social`).

### 3. Criar o App Password do Bluesky

1. Entre em [bsky.app](https://bsky.app) → **Settings → App Passwords**.
2. Crie um novo App Password (não use a senha principal da conta).
3. Anote o handle da conta (ex: `rcapitao.bsky.social`) e o App Password gerado.

### 4. Configurar variáveis e secrets no repositório GitHub

Em **Settings → Secrets and variables → Actions** deste repositório:

**Variables** (não sensíveis):
- `FEED_URL` — `https://rcapitao.com/feed.xml`
- `MASTODON_BASE_URL` — ex: `https://mastodon.social`
- `BLUESKY_HANDLE` — ex: `rcapitao.bsky.social`

**Secrets** (sensíveis):
- `MASTODON_ACCESS_TOKEN`
- `BLUESKY_APP_PASSWORD`

Se você quiser usar só uma das duas redes, basta não definir as variáveis/secrets dessa rede — o script vai ignorá-la automaticamente.

### 5. Seed inicial (evitar crosspostar todo o histórico)

Antes de ativar o agendamento, marque os posts já existentes como já publicados, sem postá-los:

1. Vá em **Actions → Crosspost new blog posts → Run workflow**.
2. Marque a opção `seed_only`.
3. Execute o workflow.

Isso registra todos os posts atuais do feed em `state.json` e faz commit automático desse arquivo. A partir daí, as execuções seguintes (agendadas ou manuais sem `seed_only`) só vão crosspostar posts realmente novos.

Alternativa local (se preferir rodar fora do GitHub Actions):

```bash
pip install -r requirements.txt
export FEED_URL="https://rcapitao.com/feed.xml"
SEED_ONLY=1 python crosspost.py
git add state.json
git commit -m "Seed crosspost state with existing posts"
git push
```

> **Migrando a URL do feed:** sempre que `FEED_URL` mudar (ex.: troca de domínio ou de caminho do feed), repita este passo de seed antes de reativar o agendamento. Como o `state.json` é indexado pelo link de cada post (`entry.link`), se os links não mudarem o histórico continua reconhecido normalmente; o seed serve como garantia extra de que nenhum post já publicado antes da migração seja crosspostado de novo.

### 6. Configurar o disparo instantâneo (repository_dispatch)

Para o `rcapitao-vhugo` conseguir acionar este workflow assim que o deploy do blog terminar, ele precisa de um token com permissão de escrever Actions neste repositório (o `GITHUB_TOKEN` padrão de um workflow só tem acesso ao próprio repositório onde roda).

1. Crie um **fine-grained personal access token** em [github.com/settings/personal-access-tokens](https://github.com/settings/personal-access-tokens/new):
   - **Repository access**: só o repositório `rcapitao/blog-crossposting`.
   - **Permissions**: `Actions` → `Read and write`.
   - Defina uma expiração (recomendado renovar periodicamente).
2. No repositório `rcapitao/rcapitao-vhugo`, vá em **Settings → Secrets and variables → Actions → New repository secret** e crie o secret `CROSSPOST_DISPATCH_TOKEN` com o valor do token gerado.

Sem esse secret configurado, o step `Notify crosspost` do `hugo.yml` falha (o deploy do site em si não é afetado) e a automação cai para o polling de 3h como único gatilho, até o secret ser configurado.

### 7. Ativar o workflow

O workflow `.github/workflows/crosspost.yml` já é acionado automaticamente pelo `repository_dispatch` do blog e roda como rede de segurança a cada 3h, das 08h às 23h (horário de Brasília), depois do push para `main`. Você também pode disparar manualmente em **Actions → Crosspost new blog posts → Run workflow**.

## Operação no dia a dia

- **Publicar um post novo no blog** já é suficiente — assim que o deploy do `rcapitao-vhugo` terminar, o `repository_dispatch` aciona o crosspost quase imediatamente. Se o disparo falhar por algum motivo, o polling de 3h (08h-23h) pega o post depois.
- **Forçar uma verificação imediata**: **Actions → Crosspost new blog posts → Run workflow** (sem marcar `seed_only`).
- **Ver o histórico de execuções e logs**: aba **Actions** do repositório. Execuções disparadas pelo blog aparecem com o evento `repository_dispatch`.
- **Confirmar o que já foi publicado**: arquivo `state.json` na raiz do repositório. Cada link aponta para um objeto indicando em quais redes já foi publicado, ex.: `{"https://rcapitao.com/posts/exemplo/": {"mastodon": true, "bluesky": true}}`.
- **Reenviar um post manualmente**: remova a entrada correspondente de `state.json` (ou só a chave da rede específica, ex. `"bluesky": true`, para reenviar somente naquela rede), faça commit/push, e execute o workflow manualmente — o post volta a ser tratado como pendente.
- **Desativar temporariamente**: em **Settings → Actions → General**, desative as Actions do repositório, ou remova/comente os gatilhos `repository_dispatch`/`schedule` no workflow.

## Troubleshooting

- **Nada é publicado**: confirme que `FEED_URL` está correto e acessível publicamente, e que pelo menos um par de credenciais (Mastodon ou Bluesky) está configurado nas secrets/variables.
- **Erro de autenticação no Mastodon**: o access token pode ter expirado ou não ter o scope `write:statuses` — gere um novo token.
- **Erro de autenticação no Bluesky**: confirme que `BLUESKY_HANDLE` é o handle completo (ex: `rcapitao.bsky.social`) e que `BLUESKY_APP_PASSWORD` é um App Password válido (não a senha da conta).
- **Posts antigos foram crosspostados de repente**: provavelmente o `state.json` foi perdido ou nunca foi seedado — repita o passo de seed inicial. Isso também acontece se o blog migrar de gerador/domínio e as URLs dos posts mudarem (o `state.json` é indexado por URL) — repita o seed depois de qualquer migração.
- **O mesmo post é publicado repetidamente no Mastodon (ou Bluesky) a cada execução**: isso era um bug conhecido (corrigido em agosto/2026) em que uma falha em uma rede fazia o post inteiro ser retentado, republicando na rede que já tinha tido sucesso. Hoje o progresso é rastreado por rede em `state.json`, então isso não deve mais acontecer — se acontecer, verifique se `state.json` está sendo commitado corretamente a cada execução (passo "Commit updated state" do workflow).
- **O workflow falha ao fazer commit do `state.json`**: confirme que a permissão `contents: write` está definida no workflow (já está por padrão neste repositório) e que não há proteção de branch bloqueando pushes diretos do `github-actions[bot]`.
- **O crosspost não roda logo depois de publicar um post**: verifique se o secret `CROSSPOST_DISPATCH_TOKEN` está configurado no `rcapitao-vhugo` e não expirou, e se o step `Notify crosspost` do `hugo.yml` terminou com sucesso na aba Actions daquele repositório. Enquanto isso não estiver funcionando, o post ainda é detectado pelo polling de 3h.

## Estrutura

- `crosspost.py` — script principal: lê o feed, decide o que é novo, publica e atualiza o estado.
- `requirements.txt` — dependências Python (`feedparser`, `requests`).
- `state.json` — registro de quais posts já foram crosspostados em cada rede (`{"url": {"mastodon": true, "bluesky": true}}`), atualizado automaticamente pelo workflow.
- `.github/workflows/crosspost.yml` — workflow do GitHub Actions, com gatilhos `repository_dispatch` (disparo instantâneo pelo blog), `schedule` (rede de segurança) e `workflow_dispatch` (incluindo a opção `seed_only`).
