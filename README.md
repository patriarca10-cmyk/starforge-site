# Starforge: Void Run — Portal de Contas & Autenticação de Pilotos

Portal web completo para criação e autenticação de contas do jogo espacial roguelite **Starforge: Void Run**, integrado diretamente ao projeto Supabase existente.

---

## 🚀 1. Como Rodar o Projeto Localmente

```bash
# 1. Instalar as dependências
npm install

# 2. Iniciar o servidor de desenvolvimento (porta 3000)
npm run dev

# 3. Para verificar tipagem e compilação
npm run lint
npm run build
```

---

## ⚙️ 2. Como Configurar a `GAME_URL`

Abra o arquivo [`src/lib/config.ts`](src/lib/config.ts):

```typescript
export const SUPABASE_URL = "https://kngcdpxofwivbaxsumrb.supabase.co";
export const SUPABASE_ANON_KEY = "sb_publishable_xRZIbD5-xCO8nSSF2K8W-Q_KN6qnY7P";

// Endereço oficial do jogo no Vercel
export const GAME_URL = "https://supreme-forc.vercel.app/";
```

Quando o piloto logar e clicar no botão dourado **JOGAR**, a aplicação irá redirecioná-lo para:
```
https://supreme-forc.vercel.app/#access_token=...&refresh_token=...
```
> **Segurança**: Os tokens de autenticação são passados **estritamente no fragmento (`#`)** da URL, nunca em query string (`?`), impedindo que fiquem registrados em logs de servidor ou histórico de proxy.

---

## 🌐 3. Como Fazer Deploy

### Deploy na Vercel:
1. Conecte o repositório deste portal na [Vercel](https://vercel.com).
2. Framework Preset: **Vite**.
3. Build Command: `npm run build`.
4. Output Directory: `dist`.
5. Se usar roteamento SPA direto por URL, crie um arquivo `vercel.json` na raiz se necessário:
```json
{
  "rewrites": [{ "source": "/(.*)", "destination": "/index.html" }]
}
```

---

## 🛠️ 4. Passo a Passo no Painel do Supabase

Acesse o painel do seu projeto no [Supabase Dashboard](https://supabase.com/dashboard/project/kngcdpxofwivbaxsumrb):

### 4.1. Configuração de URLs (`Authentication > URL Configuration`)
1. No menu lateral esquerdo, vá em **Authentication** e clique em **URL Configuration**.
2. No campo **Site URL**:
   - Defina como a URL oficial deste site de criação de conta (por exemplo: `https://starforge-conta.vercel.app` ou sua URL do AI Studio / Cloud Run).
3. Na seção **Redirect URLs** (URLs de Redirecionamento permitidas):
   - Adicione a URL deste site com curinga no final:
     `https://starforge-conta.vercel.app/*`
   - Adicione a URL local caso teste em sua máquina:
     `http://localhost:3000/*`
   - Adicione a URL do jogo hospedado no Vercel:
     `https://supreme-forc.vercel.app/*`
   - Salve as alterações clicando em **Save**.

### 4.2. Confirmação de E-mail (`Authentication > Providers > Email`)
1. No menu **Authentication**, clique em **Providers** e selecione o provedor **Email**.
2. Localize a opção **Confirm email**:
   - **Ligada (Ativada)**: O jogador cria a conta, mas **precisa abrir a caixa de entrada** e clicar no link de confirmação antes de poder logar e pilotar. O site já trata esse cenário perfeitamente exibindo a tela estelar: *"Conta criada! Enviamos um e-mail de confirmação. Confirme e depois entre para jogar."*
   - **Desligada (Desativada)**: O piloto é alistado imediatamente após o clique de cadastro, a sessão é retornada na hora e o portal o redireciona automaticamente para o Hangar (`/painel`) com o botão "JOGAR" pronto.

---

## 🎮 5. Código para Colar no Início do `main.js` do Jogo

Cole o trecho abaixo no início do arquivo de inicialização do seu jogo (`main.js`):

```javascript
// ============================================================================
// STARFORGE: VOID RUN - RECEPÇÃO DE SESSÃO AUTOMÁTICA VIA URL HASH (#)
// Cole este trecho logo após criar a instância 'supabase' no seu jogo.
// ============================================================================

async function inicializarSessaoDoPiloto(supabaseClient) {
  try {
    // 1. Lê os parâmetros contidos no fragmento (#) da URL
    const hash = window.location.hash.substring(1);
    const params = new URLSearchParams(hash);
    const accessToken = params.get('access_token');
    const refreshToken = params.get('refresh_token');

    if (accessToken && refreshToken) {
      console.log('[Starforge] Credenciais de piloto recebidas via fragmento estelar.');

      // 2. Estabelece a sessão no cliente Supabase do jogo
      const { data, error } = await supabaseClient.auth.setSession({
        access_token: accessToken,
        refresh_token: refreshToken,
      });

      if (error) {
        console.error('[Starforge] Erro ao sincronizar sessão:', error.message);
      } else {
        console.log('[Starforge] Sessão sincronizada com sucesso para o piloto:', data.user?.email);
      }

      // 3. Limpa o fragmento da barra de endereço para não expor os tokens
      window.history.replaceState(null, '', window.location.pathname + window.location.search);
    } else {
      // Verifica se já existia uma sessão salva em localStorage do jogo
      const { data: { session } } = await supabaseClient.auth.getSession();
      if (session) {
        console.log('[Starforge] Sessão pré-existente carregada da memória da nave.');
      } else {
        console.warn('[Starforge] Nenhuma sessão ativa detectada. Redirecionando para login se necessário.');
      }
    }
  } catch (err) {
    console.error('[Starforge] Falha na rotina de recepção de token:', err);
  }
}

// ----------------------------------------------------------------------------
// ONDE ENCAIXAR:
// Se o seu jogo importa @supabase/supabase-js via importmap:
// ----------------------------------------------------------------------------
// import { createClient } from '@supabase/supabase-js';
// const supabase = createClient('https://kngcdpxofwivbaxsumrb.supabase.co', 'sb_publishable_...');
//
// // CHAME A FUNÇÃO ANTES DE INICIAR O LOOP DO JOGO:
// await inicializarSessaoDoPiloto(supabase);
//
// // Em seguida, continue com o carregamento dos assets e inicialização da engine:
// iniciarJogo();
// ============================================================================
```

### Explicação do Encaixe:
1. O jogo importa `@supabase/supabase-js` via importmap e cria o cliente `supabase`.
2. Chame `await inicializarSessaoDoPiloto(supabase)` **imediatamente antes** de carregar a tela de título ou iniciar o loop do motor gráfico (Phaser, Three.js, Canvas, etc.).
3. Como `inicializarSessaoDoPiloto` é assíncrona, ela garante que o cliente Supabase do jogo esteja com a sessão autenticada pronta para ler `profiles`, `game_data` ou registrar pontuações antes da primeira nave decolar.
