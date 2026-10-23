# Documentação Técnica: Automações e Publicador Quinta Justa

Este documento registra a arquitetura, o fluxo de dados, a configuração de credenciais e a solução de erros do Publicador de Conteúdo do projeto **Quinta Justa**.

---

## 1. Arquitetura da Solução

O ecossistema é composto por dois fluxos integrados no **n8n** e uma planilha de controle no **Google Sheets**:

1. **Planilha de Controle:** [Agenda de posts · AK + Quinta Justa](https://docs.google.com/spreadsheets/d/11Nh1DrlItjI0UMGvGd0sWcDFgKnBBLYD9tF0ehR6kf4/edit)
   - Armazena a fila de publicação (datas, horários, textos, links das mídias, redes de destino e status).
   - O Publicador consome apenas linhas com `status = "aprovado"`.
   - Ao concluir, atualiza para `status = "publicado"`, preenche `publicado_em` e salva os IDs dos posts em `ids_publicados`. Em caso de falha, marca como `erro` e dispara e-mail de alerta.

2. **Repositório de Mídias:** GitHub (`jraibusiness/quinta-justa-midia`)
   - Armazena as imagens JPG dos carrosséis servidas via raw URL (`https://raw.githubusercontent.com/...`).

3. **Fluxo n8n: Publicador AK/QJ** (`workflow/qlWvjzw4ZMxwApv6`)
   - Gatilho: agendado (Schedule Trigger a cada 15 minutos).
   - Sequência de publicação:
     1. **Instagram (`IG`):** Cria os contêineres de cada lâmina (`media_type=IMAGE`), agrupa no contêiner carrossel (`media_type=CAROUSEL`) e publica.
     2. **Facebook (`FB`):** Sobe cada lâmina como foto temporária não publicada (`POST /{page-id}/photos` com `published=false`), junta os IDs e publica o carrossel no feed (`POST /{page-id}/feed` com `attached_media`).
     3. **LinkedIn (`LI`):** Baixa a 1ª imagem e cria o post no LinkedIn com texto longo.

4. **Fluxo n8n: QJ carrosséis ponte** (`workflow/XlY9McCwvowvg4Sj`)
   - Recebe webhooks locais e aciona o script Python `render.py` para gerar as lâminas e comitar no repositório.

---

## 2. Gotchas e Resolução de Erros de Credenciais (Meta / Facebook Graph API)

### O Erro
Ao tentar publicar carrossel no Facebook, o nó `FB: subir foto` falhava com o erro:
`(#200) Unpublished posts must be posted to a page as the page itself` / `Forbidden - perhaps check your credentials?`.

### A Causa
O Facebook exige que o upload de fotos não publicadas (`published=false`) seja feito **pela própria Página** e não por um usuário pessoal administrador. Se for utilizado um User Access Token, o Facebook rejeita a requisição com o código de erro `#200`.

### A Solução Definitiva
1. **Permissão Obrigatória:** O aplicativo deve ter a permissão **`pages_manage_posts`** concedida no Graph API Explorer.
2. **Page Access Token Real:** O token da página deve ser extraído do endpoint:
   ```http
   GET /me/accounts?fields=name,access_token
   ```
   Ou selecionando a página em **Page Access Tokens** no Graph API Explorer.
3. **Formatação do Token no n8n:** O token deve ser colado em linha única contínua na credencial `Meta João (página)`. Quebras de linha geram erro `400 Bad Request`.

---

## 3. Histórico de Publicações Realizadas
- `QJ-2026-10-22` (Nona · *Freude virou Freiheit*): Publicado com sucesso no Instagram (`IG:18131605258795520`) e no Facebook (`FB:2100306153564735_2043635743523970`).
- `QJ-2026-10-23` (Nona · *Quase estreou em Berlim*): Publicado com sucesso no Facebook (`FB:2100306153564735_2043636523523892`).
- **Ciclo Louise Farrenc (26/10 a 29/10/2026):**
  - `QJ-2026-10-26` (Segunda 10:00): `IG,FB,LI` · Status: `aprovado`.
  - `QJ-2026-10-27` (Terça 10:00): `IG,FB,LI` · Status: `aprovado`.
  - `QJ-2026-10-28` (Quarta 10:00): `IG,FB` · Status: `aprovado`.
  - `QJ-2026-10-29` (Quinta 10:00): `IG,FB` · Status: `aprovado`.

---

## 4. Próxima Evolução: Automação para Stories
- **Formato:** Stories exigem proporção vertical 9:16 (1080 × 1920 px).
- **API:** A API da Meta não possui "carrossel único" para Stories. Será necessário disparar 1 story por lâmina sequencialmente (`media_type=STORIES`).
- **Ação futura:** Ajustar o script `render.py` para exportar o lote em 9:16 e adicionar o ramo `Tem Stories?` no fluxo do n8n.
