# blog-crossposting

Automação que crossposta novos posts do blog [rcapitao.com](https://rcapitao.com) (gerado com [Eleventy/11ty](https://www.11ty.dev/)) para o **Mastodon** e o **Bluesky**, lendo o feed RSS do blog.

## Visão geral

A automação roda como um workflow agendado no GitHub Actions:

1. A cada 30 minutos, das 08h às 23h no horário de Brasília (ou manualmente), o workflow `.github/workflows/crosspost.yml` inicia uma máquina temporária, instala as dependências Python e executa `crosspost.py`. Fora desse intervalo (23h-08h) o workflow não roda, para não gerar execuções à toa de madrugada.
2. `crosspost.py` lê o feed RSS configurado em `FEED_URL`.
3. Compara os links dos posts do feed com os já registrados em `state.json` (o "registro do que já foi publicado").
4. Para cada post novo (do mais antigo para o mais recente), publica uma mensagem no Mastodon e no Bluesky.
5. Atualiza `state.json` com os links recém-publicados e o workflow faz commit + push automático desse arquivo no repositório.

Não há servidor rodando 24/7 nem webhook do gerador de site estático — a detecção é feita por **polling** do feed RSS.

## Formato da mensagem publicada

```
Título do post: link

Meta description do post
```

- A primeira linha junta o título e o link do post, separados por `:`.
- Há uma linha em branco entre o título/link e a meta description.
- A meta description é extraída do campo `summary`/`description` do RSS (com tags HTML removidas).
- Se o post não tiver meta description, é usado o conteúdo completo do post (campo `content` do RSS, com tags HTML removidas) na segunda parte da mensagem.
- No Bluesky, o link na primeira linha é publicado como link clicável (rich text facet), não como texto simples.
- No Bluesky, o post também inclui um card de preview do link (imagem `og:image` da página do post, título e descrição), igual ao que aparece ao colar um link manualmente. Se a página não tiver `og:image`, o post é publicado sem o card.
- Se o post não tiver meta description nem conteúdo, só é publicada a primeira linha.

## Redes suportadas

| Rede | Suportado | Autenticação |
|---|---|---|
| Mastodon | ✅ | Access token de uma aplicação OAuth |
| Bluesky | ✅ | App Password (AT Protocol) |
| Threads | ❌ (por agora) | — |

Cada rede é independente: se as variáveis/secrets de uma rede não estiverem definidas, o script simplesmente ignora essa rede e publica só nas demais (veja `post_to_mastodon` e `post_to_bluesky` em `crosspost.py`).

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

### 6. Ativar o workflow

O workflow `.github/workflows/crosspost.yml` já roda automaticamente a cada 30 minutos, das 08h às 23h (horário de Brasília), depois do push para `main`. Você também pode disparar manualmente em **Actions → Crosspost new blog posts → Run workflow**.

## Operação no dia a dia

- **Publicar um post novo no blog** já é suficiente — na próxima execução agendada (até 30 min depois, dentro do intervalo das 08h-23h) o workflow detecta e crossposta automaticamente. Se publicar fora desse intervalo, a postagem é detectada na primeira execução após as 08h.
- **Forçar uma verificação imediata**: **Actions → Crosspost new blog posts → Run workflow** (sem marcar `seed_only`).
- **Ver o histórico de execuções e logs**: aba **Actions** do repositório.
- **Confirmar o que já foi publicado**: arquivo `state.json` na raiz do repositório (lista de links já crosspostados).
- **Reenviar um post manualmente**: remova o link correspondente de `state.json`, faça commit/push, e execute o workflow manualmente — o post volta a ser tratado como novo.
- **Desativar temporariamente**: em **Settings → Actions → General**, desative as Actions do repositório, ou remova/comente o gatilho `schedule` no workflow.

## Troubleshooting

- **Nada é publicado**: confirme que `FEED_URL` está correto e acessível publicamente, e que pelo menos um par de credenciais (Mastodon ou Bluesky) está configurado nas secrets/variables.
- **Erro de autenticação no Mastodon**: o access token pode ter expirado ou não ter o scope `write:statuses` — gere um novo token.
- **Erro de autenticação no Bluesky**: confirme que `BLUESKY_HANDLE` é o handle completo (ex: `rcapitao.bsky.social`) e que `BLUESKY_APP_PASSWORD` é um App Password válido (não a senha da conta).
- **Posts antigos foram crosspostados de repente**: provavelmente o `state.json` foi perdido ou nunca foi seedado — repita o passo de seed inicial.
- **O workflow falha ao fazer commit do `state.json`**: confirme que a permissão `contents: write` está definida no workflow (já está por padrão neste repositório) e que não há proteção de branch bloqueando pushes diretos do `github-actions[bot]`.

## Estrutura

- `crosspost.py` — script principal: lê o feed, decide o que é novo, publica e atualiza o estado.
- `requirements.txt` — dependências Python (`feedparser`, `requests`).
- `state.json` — registro dos posts já crosspostados (atualizado automaticamente pelo workflow).
- `.github/workflows/crosspost.yml` — workflow agendado do GitHub Actions, com gatilho `schedule` e `workflow_dispatch` (incluindo a opção `seed_only`).
