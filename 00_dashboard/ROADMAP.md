# ZHY Storefront CRM — Roadmap & Backlog (Scrum v3.0)

> **Versão:** 3.0 — Scrum Edition
> **Última revisão:** Maio 2026
> **Metodologia:** Scrum — EPIC > STORY > TASK > SUBTASK | Sprints de 1 semana
> **Verificado contra:** Bagisto REST API v2.3.0 Shop Documentation
> **Arquitetura:** Next.js 16 (App Router) → BFF Route Handler → Bagisto (Laravel)
> **Produção:** Locaweb VPS (Ubuntu) — Docker Compose + GHCR (Veto 1)
> **Nível alvo:** Enterprise — OWASP Top 10, LGPD, observabilidade completa

> **Dev 1 — Backend/Infra/Tech Lead:** Laravel (Bagisto), Docker, GHCR, AWS S3, CI/CD, Webhooks, Banco de Dados
> **Dev 2 — Frontend/BFF:** Next.js, React, Zod, Iron Session, Route Handlers, UI/UX, SSE, Componentes

---

## Índice

1. [Repositórios](#repositorios)
2. [Arquitetura do Sistema](#arquitetura)
3. [Referência de API e Convenções](#api)
4. [🚫 Vetos Arquiteturais — Decisões Irrevogáveis](#vetos)
5. [🛡️ Auditoria de Segurança — Achados & Vulnerabilidades](#security-audit)
6. [⚡ GO LIVE — Checklist de Verificação de Base (V1–V14)](#golive)
7. [EPIC: SEC — Segurança & Hardening Enterprise](#sec)
8. [EPIC: OPS — Operações, CI/CD & Disaster Recovery](#ops)
9. [EPIC: INFRA — Infraestrutura BFF & Performance](#infra)
10. [EPIC: AUTH — Autenticação & Conta do Cliente](#auth)
11. [~~EPIC: PAY~~ — Stripe (DEPRECATED — Veto 2)](#pay)
12. [EPIC: PAY-BR — Pagamentos Brasileiros — Mercado Pago](#pay-br)
13. [EPIC: SHIP — Frete & Logística Brasil](#ship)
14. [EPIC: CAT — Catálogo & Navegação](#cat)
15. [EPIC: PROD — Produto Detalhe & Avaliações](#prod)
16. [EPIC: CART — Carrinho & Checkout](#cart)
17. [Planejamento de Sprints](#sprints)
18. [Jornadas do Cliente Mapeadas](#jornadas)
19. [Pendências e Decisões em Aberto](#pendencias)

---

## Repositórios {#repositorios}

| Repo | Branch | Responsabilidade | Dev Principal |
|---|---|---|---|
| `zhy-core-api` | `main` | Backend Laravel/Bagisto — API REST, pagamentos, catálogo, pedidos | **Dev 1** |
| `zhy-infra` | `main` | Infraestrutura Docker Compose — Nginx, MySQL, Redis, scripts de deploy | **Dev 1** |
| `zhy-storefront-crm` | `main` | Frontend Next.js 16 — storefront, BFF, componentes, SSE | **Dev 2** |

### Status atual dos repos

| Repo | Status | Bloqueadores |
|---|---|---|
| `zhy-core-api` | ⚠️ Código PIX existe, tests existem — **não verificados em staging** | `QUEUE_CONNECTION=sync` (B7), job não agendado (B8), `php artisan serve` em prod (B6) |
| `zhy-infra` | ⚠️ Nginx headers apenas no dev config | Sem prod.conf, sem restart policies, sem HTTPS (B1, B2, B3) |
| `zhy-storefront-crm` | ⚠️ Homepage, PLP, PDP, Carrinho, Checkout existem | Ver checklist V1–V14. Sem auth, sem `.env.example` (B14), sem `remotePatterns` (B13) |

---

## Arquitetura do Sistema {#arquitetura}

```
Browser
  │
  ▼
Next.js 16 App Router (zhy-storefront-crm)
  │
  ├── Server Components → lib/api.ts → http://api (Bagisto interno — server-side only)
  │
  └── Client Components → lib/shopClient.ts
        │
        ▼
      BFF: /api/shop/[...path]/route.ts   (allowlist explícita — browser nunca toca Bagisto)
        │
        ▼
      Bagisto REST API (zhy-core-api)
        │
        ├── bagisto/rest-api  → /api/v1/...   (Sanctum)
        └── Shop routes       → /api/...       (Sessão Laravel)
```

**Serviços Docker (`zhy-infra/docker-compose.yml` — dev):**

| Serviço | Imagem | Função | Redes |
|---|---|---|---|
| `proxy` | `nginx:alpine` | Reverse proxy / API gateway | `web_net` |
| `api` | `Dockerfile.dev` (zhy-core-api) | Bagisto PHP (dev: `artisan serve`) | `web_net`, `db_net` |
| `storefront` | `Dockerfile.dev` (zhy-storefront-crm) | Next.js (porta 3000) | `web_net` |
| `mysql` | `mysql:8.0` | Banco de dados | `db_net` (interno) |
| `redis` | `redis:alpine` | Cache + filas + Pub/Sub SSE | `db_net` (interno) |
| `mailpit` | `axllent/mailpit` | SMTP catcher dev (porta 8025) | `web_net` |

**Domínios locais (dev):** `app.localhost` → Bagisto Admin + API | `www.localhost` → Storefront
**Produção (Locaweb VPS):** `app.seudominio.com.br` → Bagisto | `www.seudominio.com.br` → Storefront | TLS via Let's Encrypt

---

## Referência de API e Convenções {#api}

### Dois conjuntos de rotas no Bagisto

| Prefixo | Pacote | Autenticação |
|---|---|---|
| `/api/v1/...` | `bagisto/rest-api` (Sanctum Bearer token) | Cookie httpOnly via BFF |
| `/api/...` | Shop routes internas (`packages/Webkul/Shop/src/Routes/api.php`) | Sessão Laravel |

> O BFF Next.js usa sessão Laravel (cookie `laravel_session` httpOnly). O browser **nunca** faz chamadas diretas ao Bagisto. O login via `POST /api/v1/customer/login` é executado no servidor, o token é descartado, e apenas o `Set-Cookie` é repassado ao browser.

### Endpoints — REST API v1 (`/api/v1/...`)

| Recurso | Método | Path |
|---|---|---|
| Listar categorias | GET | `/api/v1/categories` |
| Categorias descendentes | GET | `/api/v1/descendant-categories?parent_id={id}` |
| Preço máximo na categoria | GET | `/api/v1/categories/max-price/{id}` |
| Listar produtos | GET | `/api/v1/products?category_id={id}&sort={col}&order={asc\|desc}&page={n}&limit={n}` |
| Produto por ID | GET | `/api/v1/products/{id}` *(resposta inclui `variants` e `super_attributes`)* |
| Opções configuráveis | GET | `/api/v1/products/{id}/configurable-config` |
| Informação adicional | GET | `/api/v1/products/{id}/additional-information` |
| Avaliações do produto | GET | `/api/v1/products/{product_id}/reviews` |
| Criar avaliação (autenticado) | POST | `/api/v1/products/{product_id}/reviews` |
| Login | POST | `/api/v1/customer/login` |
| Registro | POST | `/api/v1/customer/register` |
| Logout | POST | `/api/v1/customer/logout` |
| Recuperar senha | POST | `/api/v1/customer/forgot-password` |
| Perfil do cliente | GET | `/api/v1/customer/get` |
| Atualizar perfil | POST | `/api/v1/customer/profile` *(com `_method=PUT`)* |
| Listar endereços | GET | `/api/v1/customer/addresses` |
| Criar endereço | POST | `/api/v1/customer/addresses` |
| Endereço padrão | PATCH | `/api/v1/customer/addresses/make-default/{id}` |
| Meus pedidos | GET | `/api/v1/customer/orders` |
| Detalhe do pedido | GET | `/api/v1/customer/orders/{id}` |
| Cancelar pedido | POST | `/api/v1/customer/orders/{id}/cancel` |
| Reordenar | GET | `/api/v1/customer/orders/reorder/{id}` |
| Faturas | GET | `/api/v1/customer/invoices` |
| Remessas | GET | `/api/v1/customer/shipments` |
| Países | GET | `/api/v1/countries` |
| Países com estados | GET | `/api/v1/countries/states/groups` |

### Endpoints — Rotas Shop internas (`/api/...`)

| Recurso | Método | Path |
|---|---|---|
| Ver carrinho | GET | `/api/checkout/cart` |
| Adicionar ao carrinho | POST | `/api/checkout/cart` |
| Atualizar quantidade | PUT | `/api/checkout/cart` |
| Esvaziar carrinho | DELETE | `/api/checkout/cart` |
| Mover item para wishlist | POST | `/api/checkout/cart/move-to-wishlist` |
| Aplicar cupom | POST | `/api/checkout/cart/coupon` |
| Remover cupom | DELETE | `/api/checkout/cart/coupon` |
| Estimar frete | POST | `/api/checkout/cart/estimate-shipping-methods` |
| Salvar endereços | POST | `/api/checkout/onepage/addresses` |
| Salvar frete | POST | `/api/checkout/onepage/shipping-methods` |
| Salvar pagamento | POST | `/api/checkout/onepage/payment-methods` |
| Realizar pedido | POST | `/api/checkout/onepage/orders` |
| Avaliações (interna) | GET | `/api/product/{id}/reviews` *(singular "product")* |
| Criar avaliação (interna) | POST | `/api/product/{id}/review` |
| Wishlist | GET | `/api/customer/wishlist` |
| Toggle wishlist | POST | `/api/customer/wishlist` |
| Wishlist → carrinho | POST | `/api/customer/wishlist/{id}/move-to-cart` |
| Remover item wishlist | DELETE | `/api/customer/wishlist/{id}` |
| Limpar wishlist | DELETE | `/api/customer/wishlist/all` |
| Árvore de categorias | GET | `/api/categories/tree` |
| Atributos de categoria | GET | `/api/categories/attributes` |
| Países (interna) | GET | `/api/core/countries` |
| Estados (interna) | GET | `/api/core/states` |

---

## 🚫 Vetos Arquiteturais — Decisões Irrevogáveis {#vetos}

> Estas decisões foram tomadas com base em análise de risco enterprise. São **irrevogáveis** e refletidas em cada EPIC, STORY e Sprint.

| # | Veto | Anti-padrão Eliminado | Decisão Definitiva |
|---|---|---|---|
| **Veto 1** | O Deploy Amador | `docker compose up --build` na VPS consome 100% CPU 3–5 min em 2 vCPUs — site fora do ar a cada deploy | **GitHub Actions compila → imagem imutável para GHCR → VPS apenas `docker pull` + rolling restart. Zero toolchain de compilação em produção.** |
| **Veto 2** | O Dual Gateway | Stripe (cartão/PIX) + Mercado Pago (boleto) = 2 webhooks, 2 credenciais, 2 reconciliações, 2 superfícies de ataque | **Mercado Pago é o gateway exclusivo para BR.** Cartão, PIX e Boleto consolidados. Stripe desabilitado. Webhook MP com HMAC-SHA256 obrigatório. |
| **Veto 3** | O DDoS Interno de PIX | `usePixPolling()` a cada 10s × 100 usuários = 18.000 requests desnecessários/h ao Bagisto | **PIX usa Server-Sent Events (SSE) + Redis Pub/Sub.** O servidor notifica o cliente exatamente uma vez, quando o webhook MP confirmar. |
| **Veto 4** | A Persistência Single Node | MySQL exclusivamente no disco da VPS → pane de hardware = perda permanente de todos os dados da loja | **`mysqldump` diário → gzip → AWS S3 (Standard-IA, KMS encryption). Retenção automática por S3 Lifecycle Policy.** |

---

## 🛡️ Auditoria de Segurança — Achados & Vulnerabilidades {#security-audit}

> Auditoria completa realizada em Maio/2026 contra OWASP Top 10. 12 vulnerabilidades encontradas. Todas endereçadas em EPIC: SEC.

### Resumo dos Achados

| # | Vulnerabilidade | Componente | OWASP | Severidade | Resolvido em |
|---|---|---|---|---|---|
| V-SEC-1 | CSRF sem proteção em todos os endpoints POST/PUT/DELETE do BFF | `src/app/api/shop/[...path]/route.ts` | A01 | 🔴 Crítico | SEC-3 (Sprint 2) |
| V-SEC-2 | Sem rate limiting no BFF — brute-force em `/customer/login` possível | `src/app/api/shop/[...path]/route.ts` | A04 | 🔴 Crítico | SEC-3 (Sprint 2) |
| V-SEC-3 | Body proxiado sem validação de schema — injeção de campos arbitrários chega ao Bagisto | `src/app/api/shop/[...path]/route.ts` | A03 | 🔴 Crítico | SEC-3 (Sprint 2) |
| V-SEC-4 | Tokens Sanctum nunca expiram — `'expiration' => null` em `config/sanctum.php` | `zhy-core-api` | A07 | 🔴 Crítico | SEC-1 (Sprint 0) |
| V-SEC-5 | `SESSION_SECURE_COOKIE` não setado — cookies via HTTP em produção | `config/session.php` | A02 | 🔴 Crítico | SEC-1 (Sprint 0) |
| V-SEC-6 | Mailpit portas 8025/1025 expostas para rede pública — emails interceptáveis | `docker-compose.yml` | A05 | 🔴 Crítico | SEC-1 (Sprint 0) |
| V-SEC-7 | CPF/CNPJ validado apenas no client-side — backend não re-valida | `src/lib/validations.ts` | A03 | 🟠 Alto | SEC-3 (Sprint 2) |
| V-SEC-8 | Sem middleware de proteção de rotas `/conta/*` | `src/middleware.ts` (ausente) | A01 | 🟠 Alto | SEC-2 (Sprint 1) |
| V-SEC-9 | Sem gerenciamento de sessão autenticada server-side | Storefront inteiro | A07 | 🟠 Alto | SEC-2 (Sprint 1) |
| V-SEC-10 | Sem validação de resposta do Bagisto — confia cegamente no shape do JSON | `src/lib/api.ts` | A08 | 🟡 Médio | SEC-3 (Sprint 2) |
| V-SEC-11 | `images.remotePatterns` ausente — `<Image>` quebra em produção | `next.config.ts` | A05 | 🟡 Médio | SEC-1 (Sprint 0) |
| V-SEC-12 | Sem `src/lib/env.ts` — app sobe com `SESSION_SECRET` vazio | Storefront inteiro | A05 | 🟡 Médio | SEC-3 (Sprint 2) |

### O que já está correto ✅

| Componente | Achado Positivo |
|---|---|
| `src/app/api/shop/[...path]/route.ts` | Allowlist explícita por endpoint + método — superfície mínima |
| `src/lib/api.ts` | Guard `typeof window !== "undefined"` — não vaza no bundle do browser |
| `src/lib/shopClient.ts` | `credentials: "same-origin"` + retry backoff exponencial para 429 |
| `zhy-core-api` | CORS estrito — origens explicitamente whitelisted, não wildcard |
| `zhy-core-api` | `StrictCorsMiddleware` — remove headers CORS se origin não na whitelist |
| `zhy-core-api` | `SESSION_HTTP_ONLY=true` — cookies não acessíveis via JavaScript |
| `zhy-infra` | MySQL e Redis em rede Docker interna (`db_net`) — não expostos externamente |
| `zhy-infra` | Nginx com `X-Frame-Options: DENY`, `X-Content-Type-Options`, `Referrer-Policy` |
| `packages/Zhy/CRM` | `throttle:10,1` no endpoint de leads |

---

## ⚡ GO LIVE — Checklist de Verificação de Base (V1–V14) {#golive}

> **⚠️ Force double-check:** Os arquivos foram auditados e **existem no repositório**. Existir ≠ funcionar em produção. Cada item precisa ser testado em staging com dados e credenciais reais antes do go-live. **Marque apenas após teste confirmado — nunca por suposição.**

| # | Feature | Arquivo verificado | Caveat / Bloqueio |
|---|---|---|---|
| V1 | Homepage com produtos reais | `src/app/page.tsx` + components | Depende de `API_URL` apontando para Bagisto de prod. Testar `FeaturedProducts` com catálogo real |
| V2 | PLP — listagem de produtos | `src/app/produtos/page.tsx` | Testar paginação, ordenação e filtros com produtos reais. Verificar ISR (`revalidate`) funcionando |
| V3 | PDP — detalhe do produto | `src/app/produtos/[slug]/page.tsx` + `AddToCartButton.tsx` | Produto configurável (variantes). Verificar `<Image>` após B13 (`remotePatterns`) |
| V4 | Carrinho (add / remove / update qty) | `src/app/carrinho/` + `src/contexts/CartContext.tsx` | Persistência entre sessões, edge cases: item sem estoque, limite de quantidade |
| V5 | Checkout multi-step | `src/app/checkout/page.tsx` | Endereço → frete → pagamento → pedido. **Depende de B17/B18 (MP não instalado)** |
| V6 | CEP autofill via ViaCEP | `src/lib/viacep.ts` | CEP válido, inválido, fora de cobertura. Verificar timeout e fallback |
| V7 | Validação CPF/CNPJ | `src/lib/validations.ts` | Dígitos verificadores inválidos, formatação automática no input |
| V8 | Página de sucesso do pedido | `src/app/pedido/sucesso/[id]/page.tsx` | Testar com pedido real pós-checkout. Guest + autenticado |
| V9 | BFF Route Handler + allowlist | `src/app/api/shop/[...path]/route.ts` | Allowlist OK. **Sem CSRF, sem rate limiting, sem Zod** — ver SEC-3. Smoke test: paths fora → 404 |
| V10 | Sitemap dinâmico + robots.txt | `src/app/sitemap.ts` + `robots.ts` | Depende de `NEXT_PUBLIC_SITE_URL` correta. Verificar `/sitemap.xml` em prod com URLs reais |
| V11 | Nginx security headers | `nginx/conf.d/default.conf` | Headers existem no config de dev. **Config de prod não existe** — bloqueia B2 |
| V12 | CI zhy-infra | `zhy-infra/.github/workflows/ci.yml` | CI cobre apenas `zhy-infra`. **Sem CI para storefront nem core-api** — ver OPS-3 |
| V13 | PIX payment | `packages/Webkul/Stripe/` + `CancelExpiredPixOrders.php` | **PIX não funciona em prod:** B7 (`QUEUE_CONNECTION=sync`) + B8 (job nunca agendado) |
| V14 | Checkout tests (Pest) | `CheckoutTest.php` + `CartTest.php` | Arquivos existem. **Executar `php artisan test --filter=Checkout` — confirmar 0 falhas antes de qualquer deploy** |

**Definition of Done — Verificação de Base:**
1. Testado em staging (não apenas `localhost`) com dados reais do Bagisto
2. Comportamento de erro testado: API fora do ar, input inválido, sessão expirada
3. Sem erros no console do browser e sem exceções em `storage/logs/laravel.log`

---

## EPIC: SEC — Segurança & Hardening Enterprise {#sec}

> **Objetivo:** Corrigir as 12 vulnerabilidades identificadas na auditoria OWASP e implementar fundação de segurança enterprise para todas as features subsequentes.
> **Prioridade:** P0 (bloqueante) — nenhuma feature nova entra em produção enquanto SEC não estiver completo.
> **Responsável principal:** Dev 1 lidera SEC-1 (Bagisto/infra). Dev 2 lidera SEC-2 e SEC-3 (Next.js/BFF).

---

### SEC-1 — Fixes Críticos Imediatos {#sec-1}

> **Sprint 0.** Correções de 30 min cada. Pré-requisito absoluto para staging funcionar.

| Task | Arquivo | Resp. | Estimativa | Bloqueador |
|---|---|---|---|---|
| SEC-1.1 — Fechar Mailpit para rede pública | `zhy-infra/docker-compose.yml` — alterar `ports: "8025:8025"` para `127.0.0.1:8025:8025` (mesmo para 1025) | **Dev 1** | 15 min | V-SEC-6 |
| SEC-1.2 — Expiração Sanctum | `zhy-core-api/config/sanctum.php` — `'expiration' => 60 * 24 * 7` (7 dias) | **Dev 1** | 15 min | V-SEC-4 |
| SEC-1.3 — Session cookie seguro | `zhy-core-api/config/session.php` — `'secure' => env('SESSION_SECURE_COOKIE', true)` | **Dev 1** | 15 min | V-SEC-5 |
| SEC-1.4 — remotePatterns no Next.js | `zhy-storefront-crm/next.config.ts` — adicionar hostname do Bagisto e CDN em `images.remotePatterns` | **Dev 2** | 30 min | V-SEC-11 |

**Implementação SEC-1.4 (next.config.ts):**

```typescript
// zhy-storefront-crm/next.config.ts
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  images: {
    remotePatterns: [
      {
        protocol: "https",
        hostname: "app.seudominio.com.br",
        pathname: "/storage/**",
      },
      {
        protocol: "http",
        hostname: "api",       // container Docker dev
        pathname: "/storage/**",
      },
    ],
  },
};

export default nextConfig;
```

**Acceptance Criteria SEC-1:**
- [ ] `docker-compose up` — porta 8025 do Mailpit não acessível de `curl http://VPS_IP:8025`
- [ ] `config/sanctum.php` com `expiration` não-nulo
- [ ] `config/session.php` com `'secure' => true` em produção
- [ ] `<Image>` do Bagisto CDN renderiza sem erros em staging

---

### SEC-2 — Iron Session: Sessão Autenticada Stateless {#sec-2}

> **Sprint 1.** Responsável: **Dev 2.** Sem isso, não há auth, não há `/conta`, não há checkout autenticado.

| Story | Arquivo | Resp. | Estimativa |
|---|---|---|---|
| SEC-2.1 — Interface de sessão + helpers | `src/lib/session.ts` | **Dev 2** | 1h |
| SEC-2.2 — Route Handler login | `src/app/api/auth/login/route.ts` | **Dev 2** | 2h |
| SEC-2.3 — Route Handler me | `src/app/api/auth/me/route.ts` | **Dev 2** | 30 min |
| SEC-2.4 — Route Handler logout | `src/app/api/auth/logout/route.ts` | **Dev 2** | 30 min |
| SEC-2.5 — Middleware de proteção de rotas | `src/middleware.ts` | **Dev 2** | 1h |
| SEC-2.6 — Páginas de auth UI | `src/app/login/`, `src/app/register/`, `src/app/esqueci-senha/` | **Dev 2** | 8h |

**Implementação SEC-2.1 (`src/lib/session.ts`):**

```typescript
import { getIronSession, SessionOptions } from "iron-session";
import { cookies } from "next/headers";
import { redirect } from "next/navigation";

export interface SessionData {
  customerId?: number;
  email?: string;
  firstName?: string;
  bagistoSessionCookie?: string; // 'laravel_session' value
  isLoggedIn: boolean;
}

export const SESSION_OPTIONS: SessionOptions = {
  password: process.env.SESSION_SECRET!,
  cookieName: "zhy_session",
  cookieOptions: {
    secure: process.env.NODE_ENV === "production",
    httpOnly: true,
    sameSite: "strict",
    maxAge: 60 * 60 * 24 * 7, // 7 dias
  },
};

export async function getSession() {
  return getIronSession<SessionData>(await cookies(), SESSION_OPTIONS);
}

export async function requireAuth() {
  const session = await getSession();
  if (!session.isLoggedIn) redirect("/login");
  return session;
}
```

**Implementação SEC-2.2 (`src/app/api/auth/login/route.ts`):**

```typescript
import { NextRequest, NextResponse } from "next/server";
import { z } from "zod";
import { getSession } from "@/lib/session";

const LoginSchema = z.object({
  email: z.string().email(),
  password: z.string().min(8),
});

export async function POST(request: NextRequest) {
  const body = await request.json().catch(() => null);
  const parsed = LoginSchema.safeParse(body);
  if (!parsed.success) {
    return NextResponse.json({ error: "Dados inválidos" }, { status: 400 });
  }

  const bagistoRes = await fetch(
    `${process.env.INTERNAL_API_URL}/api/v1/customer/login`,
    {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(parsed.data),
    }
  );

  if (!bagistoRes.ok) {
    const err = await bagistoRes.json().catch(() => ({}));
    return NextResponse.json({ error: err.message ?? "Login falhou" }, { status: 401 });
  }

  const data = await bagistoRes.json();
  const bagistoSetCookie = bagistoRes.headers.get("set-cookie");
  const laravelSession = bagistoSetCookie?.match(/laravel_session=([^;]+)/)?.[1];

  const session = await getSession();
  session.customerId = data.data.id;
  session.email = data.data.email;
  session.firstName = data.data.first_name;
  session.bagistoSessionCookie = laravelSession;
  session.isLoggedIn = true;
  await session.save();

  return NextResponse.json({ ok: true, firstName: session.firstName });
}
```

**Implementação SEC-2.3 (`src/app/api/auth/me/route.ts`):**

```typescript
import { NextResponse } from "next/server";
import { getSession } from "@/lib/session";

export async function GET() {
  const session = await getSession();
  if (!session.isLoggedIn) {
    return NextResponse.json({ isLoggedIn: false }, { status: 401 });
  }
  return NextResponse.json({
    isLoggedIn: true,
    customerId: session.customerId,
    email: session.email,
    firstName: session.firstName,
  });
}
```

**Implementação SEC-2.4 (`src/app/api/auth/logout/route.ts`):**

```typescript
import { NextResponse } from "next/server";
import { getSession } from "@/lib/session";

export async function POST() {
  const session = await getSession();
  session.destroy();
  return NextResponse.json({ ok: true });
}
```

**Implementação SEC-2.5 (`src/middleware.ts`):**

```typescript
import { NextRequest, NextResponse } from "next/server";
import { getIronSession } from "iron-session";
import { SessionData, SESSION_OPTIONS } from "@/lib/session";

const PROTECTED_PREFIXES = ["/conta", "/checkout", "/pedido"];
const PUBLIC_AUTH_PATHS = ["/login", "/register", "/esqueci-senha"];

export async function middleware(request: NextRequest) {
  const { pathname } = request.nextUrl;

  const isProtected = PROTECTED_PREFIXES.some((p) => pathname.startsWith(p));
  const isPublicAuth = PUBLIC_AUTH_PATHS.some((p) => pathname.startsWith(p));

  if (!isProtected && !isPublicAuth) return NextResponse.next();

  const response = NextResponse.next();
  const session = await getIronSession<SessionData>(
    request.cookies as any,
    SESSION_OPTIONS
  );

  if (isProtected && !session.isLoggedIn) {
    const loginUrl = new URL("/login", request.url);
    loginUrl.searchParams.set("next", pathname);
    return NextResponse.redirect(loginUrl);
  }

  if (isPublicAuth && session.isLoggedIn) {
    return NextResponse.redirect(new URL("/conta", request.url));
  }

  return response;
}

export const config = {
  matcher: ["/((?!api|_next/static|_next/image|favicon.ico).*)"],
};
```

**SEC-2.6 — Páginas de Auth:**

| Página | Path | Resp. | Estimativa | Notas |
|---|---|---|---|---|
| Login | `src/app/login/page.tsx` | **Dev 2** | 2h | Form → `POST /api/auth/login` → redirect `/conta` ou `?next=` |
| Registro | `src/app/register/page.tsx` | **Dev 2** | 2h | Form → `POST /api/shop/v1/customer/register` via BFF |
| Esqueci senha | `src/app/esqueci-senha/page.tsx` | **Dev 2** | 1h | Form → `POST /api/shop/v1/customer/forgot-password` via BFF |
| Redefinir senha | `src/app/redefinir-senha/page.tsx` | **Dev 2** | 1h | Token via query param |
| UI/UX: ZHY Visual Identity | Todos os formulários de auth | **Dev 2** | 2h | `bg-[#F5EFE5]` cream, `text-[#1F2430]` navy, `font-title` Caslon Ionic, glassmorphism |

**Acceptance Criteria SEC-2:**
- [ ] `GET /api/auth/me` retorna 401 sem cookie
- [ ] Login bem-sucedido → `zhy_session` cookie httpOnly criado
- [ ] `/conta` sem sessão → redirect `/login?next=/conta`
- [ ] Logout → cookie destruído → `/conta` redireciona novamente

---

### SEC-3 — BFF Hardening: CSRF, Rate Limiting, Zod, env.ts {#sec-3}

> **Sprint 2.** Responsável: **Dev 2.** Resolve V-SEC-1, V-SEC-2, V-SEC-3, V-SEC-7, V-SEC-10, V-SEC-12.

| Story | Arquivo | Resp. | Estimativa |
|---|---|---|---|
| SEC-3.1 — CSRF double-submit cookie | `src/app/api/auth/csrf/route.ts` + handler BFF | **Dev 2** | 3h |
| SEC-3.2 — Rate limiter (memory) | `src/lib/rateLimiter.ts` | **Dev 2** | 2h |
| SEC-3.3 — Zod validation nos endpoints críticos | `src/lib/schemas/checkout.ts` | **Dev 2** | 3h |
| SEC-3.4 — Validação de variáveis de ambiente | `src/lib/env.ts` | **Dev 2** | 1h |
| SEC-3.5 — `.env.example` completo | `zhy-storefront-crm/.env.example` | **Dev 2** | 30 min |

**Implementação SEC-3.1 (CSRF):**

```typescript
// src/app/api/auth/csrf/route.ts
import { NextResponse } from "next/server";
import { randomBytes } from "crypto";

export async function GET() {
  const token = randomBytes(32).toString("hex");
  const response = NextResponse.json({ csrfToken: token });
  response.cookies.set("csrf_token", token, {
    httpOnly: false, // lido pelo JS para incluir no header
    sameSite: "strict",
    secure: process.env.NODE_ENV === "production",
    maxAge: 60 * 60, // 1 hora
  });
  return response;
}
```

```typescript
// Validação no BFF handler — src/app/api/shop/[...path]/route.ts (adição)
const csrfHeader = request.headers.get("x-csrf-token");
const csrfCookie = request.cookies.get("csrf_token")?.value;
const MUTATION_METHODS = ["POST", "PUT", "PATCH", "DELETE"];

if (MUTATION_METHODS.includes(request.method)) {
  if (!csrfHeader || !csrfCookie || csrfHeader !== csrfCookie) {
    return NextResponse.json({ error: "CSRF token inválido" }, { status: 403 });
  }
}
```

**Implementação SEC-3.2 (Rate Limiter):**

```typescript
// src/lib/rateLimiter.ts
interface RateLimitEntry { count: number; resetAt: number; }
const store = new Map<string, RateLimitEntry>();

export function createRateLimiter(max: number, windowMs: number) {
  return function check(key: string): { allowed: boolean; remaining: number } {
    const now = Date.now();
    const entry = store.get(key);

    if (!entry || now > entry.resetAt) {
      store.set(key, { count: 1, resetAt: now + windowMs });
      return { allowed: true, remaining: max - 1 };
    }

    if (entry.count >= max) return { allowed: false, remaining: 0 };

    entry.count++;
    return { allowed: true, remaining: max - entry.count };
  };
}

// Instâncias prontas para uso:
export const authRateLimiter    = createRateLimiter(5, 15 * 60 * 1000);  // 5 req / 15 min (login/register)
export const bffRateLimiter     = createRateLimiter(100, 60 * 1000);     // 100 req / min (BFF geral)
export const ordersRateLimiter  = createRateLimiter(5, 60 * 60 * 1000);  // 5 pedidos / hora (anti-fraude)
```

**Implementação SEC-3.3 (Zod Schemas):**

```typescript
// src/lib/schemas/checkout.ts
import { z } from "zod";

export const AddressSchema = z.object({
  first_name:  z.string().min(1).max(100),
  last_name:   z.string().min(1).max(100),
  phone:       z.string().regex(/^(\+55)?\d{10,11}$/),
  address1:    z.array(z.string().min(1).max(200)),
  city:        z.string().min(1).max(100),
  state:       z.string().length(2),  // UF código ISO
  postcode:    z.string().regex(/^\d{8}$/),
  country:     z.literal("BR"),
  cpf_cnpj:    z.string().regex(/^(\d{11}|\d{14})$/).optional(),
});

export const CartItemSchema = z.object({
  product_id: z.number().int().positive(),
  quantity:   z.number().int().positive().max(99),
  super_attribute: z.record(z.string(), z.string()).optional(),
});

export const CouponSchema = z.object({
  code: z.string().min(1).max(50),
});
```

**Implementação SEC-3.4 (env.ts):**

```typescript
// src/lib/env.ts
import { createEnv } from "@t3-oss/env-nextjs";
import { z } from "zod";

export const env = createEnv({
  server: {
    SESSION_SECRET:    z.string().min(32),
    INTERNAL_API_URL:  z.string().url(),
    NODE_ENV:          z.enum(["development", "production", "test"]),
  },
  client: {
    NEXT_PUBLIC_API_URL:   z.string().url(),
    NEXT_PUBLIC_SITE_URL:  z.string().url(),
    NEXT_PUBLIC_MP_PUBLIC_KEY: z.string().min(1),
  },
  runtimeEnv: {
    SESSION_SECRET:            process.env.SESSION_SECRET,
    INTERNAL_API_URL:          process.env.INTERNAL_API_URL,
    NODE_ENV:                  process.env.NODE_ENV,
    NEXT_PUBLIC_API_URL:       process.env.NEXT_PUBLIC_API_URL,
    NEXT_PUBLIC_SITE_URL:      process.env.NEXT_PUBLIC_SITE_URL,
    NEXT_PUBLIC_MP_PUBLIC_KEY: process.env.NEXT_PUBLIC_MP_PUBLIC_KEY,
  },
});
```

**SEC-3.5 (`.env.example`):**

```bash
# zhy-storefront-crm/.env.example

# === Sessão (gerar com: openssl rand -hex 32) ===
SESSION_SECRET=your-32-char-minimum-secret-here

# === URLs internas (Docker) ===
INTERNAL_API_URL=http://api  # Bagisto container — server-side only

# === URLs públicas ===
NEXT_PUBLIC_API_URL=https://app.seudominio.com.br
NEXT_PUBLIC_SITE_URL=https://www.seudominio.com.br

# === Mercado Pago ===
NEXT_PUBLIC_MP_PUBLIC_KEY=APP_USR-XXXX      # public — ok no browser
MP_ACCESS_TOKEN=APP_USR-XXXX               # PRIVADO — nunca no NEXT_PUBLIC_

# === Opcional: Sentry ===
NEXT_PUBLIC_SENTRY_DSN=https://xxxx@sentry.io/xxxx
```

**Acceptance Criteria SEC-3:**
- [ ] POST sem `x-csrf-token` retorna 403
- [ ] 6º login attempt em 15 min retorna 429
- [ ] Body com campo não previsto no Zod schema → 400 antes de chegar no Bagisto
- [ ] App não sobe se `SESSION_SECRET` não definido (env.ts lança erro)
- [ ] `.env.example` tem todas as variáveis usadas em `env.ts`

---

### SEC-4 — LGPD Compliance {#sec-4}

> **Sprint 7+ (pós-launch prioritário).** Responsável: **Dev 1 (backend)** + **Dev 2 (frontend)**.

| Task | Arquivo | Resp. | Estimativa |
|---|---|---|---|
| SEC-4.1 — Solicitação de exclusão de dados | `src/app/conta/privacidade/page.tsx` + API endpoint Bagisto | **Dev 2** + **Dev 1** | 4h |
| SEC-4.2 — Exportar dados pessoais (LGPD Art. 18) | Export CSV via `GET /api/v1/customer/export` (custom endpoint) | **Dev 1** | 4h |
| SEC-4.3 — Cookie consent banner | `src/components/CookieConsent.tsx` (localStorage, sem Google Analytics de terceiros) | **Dev 2** | 3h |

**Acceptance Criteria SEC-4:**
- [ ] Cliente consegue solicitar exclusão de conta via UI
- [ ] Export de dados gera arquivo com todos os campos PII
- [ ] Cookie banner aparece na primeira visita e respeita preferência

---
## EPIC: OPS — Operações, CI/CD & Disaster Recovery {#ops}

> **Objetivo:** Infraestrutura de produção enterprise — Docker imutável, CI/CD sem downtime, observabilidade e backup automatizado.
> **Responsável principal:** Dev 1 (toda infra Bagisto/Docker/AWS). Dev 2 colabora em CI storefront e Sentry no frontend.

---

### OPS-1 — Infraestrutura de Produção Docker {#ops-1}

> **Sprint 4.** Responsável: **Dev 1.** Predecessor: SEC-1 completo.

| Story | Arquivo | Resp. | Estimativa |
|---|---|---|---|
| OPS-1.1 — docker-compose.prod.yml | `zhy-infra/docker-compose.prod.yml` | **Dev 1** | 4h |
| OPS-1.2 — supervisor.conf | `zhy-core-api/supervisor.conf` | **Dev 1** | 2h |
| OPS-1.3 — Dockerfile.prod (Bagisto) | `zhy-core-api/Dockerfile.prod` | **Dev 1** | 3h |
| OPS-1.4 — Dockerfile.prod (Storefront) | `zhy-storefront-crm/Dockerfile.prod` | **Dev 2** | 2h |
| OPS-1.5 — Nginx prod.conf (HTTPS + rate limit) | `zhy-infra/nginx/conf.d/prod.conf` | **Dev 1** | 3h |
| OPS-1.6 — Certbot / Let's Encrypt | Setup manual VPS + `cron` mensal | **Dev 1** | 1h |

**OPS-1.1 — docker-compose.prod.yml:**

```yaml
# zhy-infra/docker-compose.prod.yml
version: "3.9"

services:
  proxy:
    image: nginx:1.27-alpine
    ports: ["80:80", "443:443"]
    volumes:
      - ./nginx/conf.d/prod.conf:/etc/nginx/conf.d/default.conf:ro
      - /etc/letsencrypt:/etc/letsencrypt:ro
      - /var/www/certbot:/var/www/certbot:ro
    depends_on: [api, storefront]
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "nginx", "-t"]
      interval: 30s
      retries: 3
    mem_limit: 128m
    networks: [web_net]

  api:
    image: ghcr.io/mzgui/zhy-core-api:${API_TAG:-latest}
    env_file: .env.api.prod
    depends_on:
      mysql:
        condition: service_healthy
      redis:
        condition: service_healthy
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/api/v1/status"]
      interval: 30s
      retries: 3
    mem_limit: 512m
    networks: [web_net, db_net]

  storefront:
    image: ghcr.io/mzgui/zhy-storefront-crm:${STOREFRONT_TAG:-latest}
    env_file: .env.storefront.prod
    depends_on: [api]
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/api/health"]
      interval: 30s
      retries: 3
    mem_limit: 512m
    networks: [web_net]

  mysql:
    image: mysql:8.0
    env_file: .env.mysql.prod
    volumes:
      - mysql_data:/var/lib/mysql
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      retries: 5
    mem_limit: 1g
    networks: [db_net]

  redis:
    image: redis:7-alpine
    command: redis-server --requirepass ${REDIS_PASSWORD} --maxmemory 256mb --maxmemory-policy allkeys-lru
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      retries: 3
    mem_limit: 256m
    networks: [db_net]

volumes:
  mysql_data:

networks:
  web_net:
  db_net:
    internal: true   # MySQL e Redis inacessíveis de fora
```

**OPS-1.2 — supervisor.conf:**

```ini
; zhy-core-api/supervisor.conf
[supervisord]
nodaemon=true
logfile=/dev/stdout
logfile_maxbytes=0

[program:php-fpm]
command=php-fpm -F
autostart=true
autorestart=true
stdout_logfile=/dev/stdout
stdout_logfile_maxbytes=0
stderr_logfile=/dev/stderr
stderr_logfile_maxbytes=0

[program:queue-worker]
command=php /var/www/html/artisan queue:work redis --sleep=3 --tries=3 --max-time=3600 --memory=128
autostart=true
autorestart=true
numprocs=2
process_name=%(program_name)s_%(process_num)02d
stdout_logfile=/dev/stdout
stdout_logfile_maxbytes=0

[program:scheduler]
command=/bin/sh -c "while true; do php /var/www/html/artisan schedule:run --no-interaction && sleep 60; done"
autostart=true
autorestart=true
stdout_logfile=/dev/stdout
stdout_logfile_maxbytes=0
```

**OPS-1.3 — Dockerfile.prod (Bagisto):**

```dockerfile
# zhy-core-api/Dockerfile.prod
FROM php:8.3-fpm-alpine AS base

RUN apk add --no-cache \
    supervisor nginx curl \
    libpng-dev libjpeg-dev freetype-dev \
    icu-dev libzip-dev oniguruma-dev \
    && docker-php-ext-configure gd --with-freetype --with-jpeg \
    && docker-php-ext-install -j$(nproc) \
       pdo_mysql gd mbstring intl zip bcmath opcache pcntl

COPY --from=composer:2 /usr/bin/composer /usr/bin/composer

WORKDIR /var/www/html
COPY . .

RUN composer install --no-dev --optimize-autoloader --no-interaction \
    && php artisan config:cache \
    && php artisan route:cache \
    && php artisan view:cache \
    && php artisan event:cache \
    && chown -R www-data:www-data storage bootstrap/cache \
    && chmod -R 775 storage bootstrap/cache

COPY supervisor.conf /etc/supervisor/conf.d/zhy.conf
EXPOSE 9000
CMD ["supervisord", "-c", "/etc/supervisor/conf.d/zhy.conf"]
```

**OPS-1.5 — Nginx prod.conf:**

```nginx
# zhy-infra/nginx/conf.d/prod.conf
limit_req_zone $binary_remote_addr zone=api_limit:10m rate=60r/m;
limit_req_zone $binary_remote_addr zone=auth_limit:10m rate=5r/m;

# HTTP → HTTPS redirect
server {
    listen 80;
    server_name app.seudominio.com.br www.seudominio.com.br;
    location /.well-known/acme-challenge/ { root /var/www/certbot; }
    location / { return 301 https://$host$request_uri; }
}

# Bagisto API (HTTPS)
server {
    listen 443 ssl http2;
    server_name app.seudominio.com.br;
    ssl_certificate     /etc/letsencrypt/live/app.seudominio.com.br/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/app.seudominio.com.br/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    add_header X-Frame-Options DENY;
    add_header X-Content-Type-Options nosniff;
    add_header Referrer-Policy strict-origin-when-cross-origin;
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    location /api/v1/customer/login {
        limit_req zone=auth_limit burst=3 nodelay;
        proxy_pass http://api:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /api/ {
        limit_req zone=api_limit burst=20 nodelay;
        proxy_pass http://api:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

# Storefront (HTTPS)
server {
    listen 443 ssl http2;
    server_name www.seudominio.com.br;
    ssl_certificate     /etc/letsencrypt/live/www.seudominio.com.br/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/www.seudominio.com.br/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;

    add_header X-Frame-Options DENY;
    add_header X-Content-Type-Options nosniff;
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    location / {
        proxy_pass http://storefront:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

**Acceptance Criteria OPS-1:**
- [ ] `docker compose -f docker-compose.prod.yml up -d` sem erros
- [ ] Queue worker processando jobs (verificar `horizon` ou `queue:monitor`)
- [ ] Scheduler rodando `CancelExpiredPixOrders` a cada minuto
- [ ] HTTPS com TLS 1.2+ em ambos os domínios
- [ ] MySQL e Redis inacessíveis externamente

---

### OPS-2 — Observabilidade: Sentry + Pino + Health {#ops-2}

> **Sprint 4.** Responsável: **Dev 1** (Bagisto Sentry) + **Dev 2** (Next.js Sentry + Pino + health endpoint).

| Story | Arquivo | Resp. | Estimativa |
|---|---|---|---|
| OPS-2.1 — Sentry no Bagisto | `zhy-core-api/config/sentry.php` + `.env` | **Dev 1** | 2h |
| OPS-2.2 — Sentry + Pino no Next.js | `src/lib/logger.ts` + `sentry.client.config.ts` | **Dev 2** | 2h |
| OPS-2.3 — Health check endpoint Next.js | `src/app/api/health/route.ts` | **Dev 2** | 30 min |
| OPS-2.4 — Health check endpoint Bagisto | `routes/api.php` — `/api/v1/status` | **Dev 1** | 30 min |

```typescript
// src/app/api/health/route.ts
import { NextResponse } from "next/server";

export async function GET() {
  return NextResponse.json({
    status: "ok",
    timestamp: new Date().toISOString(),
    version: process.env.npm_package_version ?? "unknown",
  });
}
```

```typescript
// src/lib/logger.ts — Pino structured logging
import pino from "pino";

export const logger = pino({
  level: process.env.LOG_LEVEL ?? "info",
  transport: process.env.NODE_ENV !== "production"
    ? { target: "pino-pretty", options: { colorize: true } }
    : undefined,
});
```

---

### OPS-3 — CI/CD: GitHub Actions + GHCR + deploy.sh [Veto 1] {#ops-3}

> **Sprint 4.** Responsável: **Dev 1** (CD pipelines + deploy.sh). **Dev 2** colabora em CI storefront.
> **Rationale Veto 1:** VPS tem 2 vCPUs — `docker build` de imagem Next.js consome 100% CPU por 5+ min. Site inacessível durante deploy. Solução: compilar no GitHub Actions e fazer push para GHCR. VPS apenas faz `docker pull` da imagem imutável.

| Story | Arquivo | Resp. | Estimativa |
|---|---|---|---|
| OPS-3.1 — CI storefront (lint, build, test) | `zhy-storefront-crm/.github/workflows/ci.yml` | **Dev 2** | 3h |
| OPS-3.2 — CI core-api (PHPUnit, Pint) | `zhy-core-api/.github/workflows/ci.yml` | **Dev 1** | 3h |
| OPS-3.3 — CD core-api → GHCR | `zhy-core-api/.github/workflows/cd.yml` | **Dev 1** | 2h |
| OPS-3.4 — CD storefront → GHCR | `zhy-storefront-crm/.github/workflows/cd.yml` | **Dev 2** | 2h |
| OPS-3.5 — deploy.sh no VPS | `zhy-infra/scripts/deploy.sh` | **Dev 1** | 2h |

**OPS-3.3 — CD core-api (GHCR):**

```yaml
# zhy-core-api/.github/workflows/cd.yml
name: CD — Build & Push to GHCR (core-api)
on:
  push:
    branches: [main]

jobs:
  build-push:
    runs-on: ubuntu-latest
    permissions:
      packages: write
      contents: read
    steps:
      - uses: actions/checkout@v4

      - name: Log in to GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          file: Dockerfile.prod
          push: true
          tags: |
            ghcr.io/mzgui/zhy-core-api:latest
            ghcr.io/mzgui/zhy-core-api:${{ github.sha }}
```

**OPS-3.5 — deploy.sh [Veto 1]:**

```bash
#!/usr/bin/env bash
# zhy-infra/scripts/deploy.sh
# Executa via GitHub Actions Self-Hosted Runner OU chamado manualmente via SSH
# NUNCA faz build — apenas pull de imagem GHCR já compilada (Veto 1)
set -euo pipefail

COMPOSE_FILE="/opt/zhy/docker-compose.prod.yml"
REGISTRY="ghcr.io"
API_IMAGE="ghcr.io/mzgui/zhy-core-api"
STOREFRONT_IMAGE="ghcr.io/mzgui/zhy-storefront-crm"

echo "==> Autenticando no GHCR..."
echo "${GHCR_PAT}" | docker login ghcr.io -u mzgui --password-stdin

echo "==> Pulling imagens novas (sem build)..."
docker pull "${API_IMAGE}:latest"
docker pull "${STOREFRONT_IMAGE}:latest"

echo "==> Rolling restart (sem downtime)..."
docker compose -f "${COMPOSE_FILE}" up -d --no-build --remove-orphans

echo "==> Health check..."
MAX_RETRIES=10
RETRY=0
until curl -sf http://localhost/api/v1/status > /dev/null; do
  RETRY=$((RETRY + 1))
  if [ $RETRY -ge $MAX_RETRIES ]; then
    echo "ERRO: Health check falhou após ${MAX_RETRIES} tentativas. Revertendo..."
    docker compose -f "${COMPOSE_FILE}" rollback 2>/dev/null || \
      docker compose -f "${COMPOSE_FILE}" restart api storefront
    exit 1
  fi
  echo "  Aguardando... tentativa ${RETRY}/${MAX_RETRIES}"
  sleep 5
done

echo "==> Deploy concluído com sucesso."
```

**Acceptance Criteria OPS-3:**
- [ ] Push para `main` no core-api dispara build e push para GHCR
- [ ] Imagem tagged com `latest` + `sha` do commit
- [ ] VPS não tem Dockerfile nem `npm install` — apenas `docker pull`
- [ ] `deploy.sh` faz rollback automático se health check falhar após 50s

---

### OPS-4 — Disaster Recovery: MySQL → S3 [Veto 4] {#ops-4}

> **Sprint 4.** Responsável: **Dev 1.**
> **Rationale Veto 4:** VPS tem 1 disco. Falha de hardware = perda de 100% dos dados. Solução: backup diário automatizado para S3 com retenção gerenciada por lifecycle policy.

| Story | Arquivo | Resp. | Estimativa |
|---|---|---|---|
| OPS-4.1 — mysql-backup.sh | `zhy-infra/scripts/mysql-backup.sh` | **Dev 1** | 3h |
| OPS-4.2 — Cron job no VPS | `/etc/cron.d/zhy-mysql-backup` | **Dev 1** | 30 min |
| OPS-4.3 — S3 Lifecycle Policy | `zhy-infra/s3-lifecycle-policy.json` | **Dev 1** | 1h |
| OPS-4.4 — IAM Policy mínima | `zhy-infra/iam-backup-policy.json` | **Dev 1** | 30 min |
| OPS-4.5 — Teste de restore documentado | `zhy-infra/RESTORE.md` | **Dev 1** | 1h |

**OPS-4.1 — mysql-backup.sh [Veto 4]:**

```bash
#!/usr/bin/env bash
# zhy-infra/scripts/mysql-backup.sh
# Executado diariamente via cron às 02:00 UTC
set -euo pipefail

TIMESTAMP=$(date +%Y%m%d_%H%M%S)
DATE=$(date +%Y%m%d)
DAY_OF_WEEK=$(date +%u)
DAY_OF_MONTH=$(date +%d)
BACKUP_DIR="/tmp/mysql-backups"
BACKUP_FILE="${BACKUP_DIR}/zhy_${TIMESTAMP}.sql.gz"
S3_BUCKET="s3://zhy-backups-prod"

# Classificação automática de retenção
if [ "${DAY_OF_MONTH}" = "01" ]; then
  S3_PREFIX="monthly"
elif [ "${DAY_OF_WEEK}" = "7" ]; then
  S3_PREFIX="weekly"
else
  S3_PREFIX="daily"
fi

S3_PATH="${S3_BUCKET}/${S3_PREFIX}/zhy_${TIMESTAMP}.sql.gz"

mkdir -p "${BACKUP_DIR}"

echo "[$(date -u +%Y-%m-%dT%H:%M:%SZ)] Iniciando backup MySQL → ${S3_PREFIX}..."

# Dump com lock mínimo (InnoDB)
mysqldump \
  --host=mysql \
  --user="${MYSQL_USER}" \
  --password="${MYSQL_PASSWORD}" \
  --single-transaction \
  --routines \
  --triggers \
  --set-gtid-purged=OFF \
  "${MYSQL_DATABASE}" | gzip -9 > "${BACKUP_FILE}"

BACKUP_SIZE=$(du -sh "${BACKUP_FILE}" | cut -f1)
echo "  Backup local: ${BACKUP_FILE} (${BACKUP_SIZE})"

# Verificação de integridade
if ! gzip -t "${BACKUP_FILE}"; then
  echo "ERRO: Arquivo gzip corrompido. Abortando upload."
  rm -f "${BACKUP_FILE}"
  exit 1
fi

# Upload para S3 com KMS encryption (Veto 4)
aws s3 cp "${BACKUP_FILE}" "${S3_PATH}" \
  --storage-class STANDARD_IA \
  --sse aws:kms \
  --region us-east-1

echo "  Upload S3 concluído: ${S3_PATH}"
rm -f "${BACKUP_FILE}"
echo "[$(date -u +%Y-%m-%dT%H:%M:%SZ)] Backup concluído com sucesso."
```

**OPS-4.2 — Cron:**

```cron
# /etc/cron.d/zhy-mysql-backup
# Backup diário às 02:00 UTC
SHELL=/bin/bash
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

0 2 * * * root /opt/zhy/scripts/mysql-backup.sh >> /var/log/zhy-mysql-backup.log 2>&1
```

**OPS-4.3 — S3 Lifecycle Policy:**

```json
{
  "Rules": [
    {
      "ID": "delete-daily-after-7-days",
      "Status": "Enabled",
      "Filter": { "Prefix": "daily/" },
      "Expiration": { "Days": 7 }
    },
    {
      "ID": "delete-weekly-after-30-days",
      "Status": "Enabled",
      "Filter": { "Prefix": "weekly/" },
      "Expiration": { "Days": 30 }
    },
    {
      "ID": "delete-monthly-after-365-days",
      "Status": "Enabled",
      "Filter": { "Prefix": "monthly/" },
      "Expiration": { "Days": 365 }
    }
  ]
}
```

**OPS-4.4 — IAM Policy mínima (princípio do menor privilégio):**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowBackupWrite",
      "Effect": "Allow",
      "Action": ["s3:PutObject", "s3:GetObject"],
      "Resource": "arn:aws:s3:::zhy-backups-prod/*"
    },
    {
      "Sid": "AllowListBucket",
      "Effect": "Allow",
      "Action": ["s3:ListBucket"],
      "Resource": "arn:aws:s3:::zhy-backups-prod"
    },
    {
      "Sid": "AllowKMSEncryption",
      "Effect": "Allow",
      "Action": ["kms:GenerateDataKey", "kms:Decrypt"],
      "Resource": "arn:aws:kms:us-east-1:ACCOUNT_ID:key/KEY_ID"
    }
  ]
}
```

**Acceptance Criteria OPS-4:**
- [ ] Cron job dispara às 02:00 UTC e log registra sucesso
- [ ] Arquivo `.sql.gz` sobe para S3 com `STANDARD_IA` e KMS
- [ ] Lifecycle policy aplicada no bucket — arquivos daily expiram em 7 dias
- [ ] Restore testado: `gzip -d backup.sql.gz | mysql -u root -p zhy_prod` funciona
- [ ] IAM user tem apenas as permissões listadas — nada mais

---
## EPIC: INFRA — Infraestrutura BFF & Performance {#infra}

> **Objetivo:** Fundação técnica do BFF (Next.js Route Handler), performance (ISR, cache), e configurações essenciais de produção.
> **Responsável:** Dev 2 (todos os arquivos `zhy-storefront-crm`), exceto INFRA-3 (Dev 1 side).

| ID | Story | Arquivo | Resp. | Estimativa | Sprint |
|---|---|---|---|---|---|
| INFRA-1 | Extend BFF allowlist — novos endpoints | `src/app/api/shop/[...path]/route.ts` | **Dev 2** | 2h | Sprint 3 |
| INFRA-2 | `shopClient.ts` — retry + backoff | `src/lib/shopClient.ts` | **Dev 2** | 2h | Sprint 1 |
| INFRA-3 | Variáveis de ambiente Bagisto em prod | `zhy-core-api/.env.prod.example` | **Dev 1** | 1h | Sprint 4 |
| INFRA-4 | ISR revalidation tuning | `src/app/produtos/page.tsx`, `src/app/produtos/[slug]/page.tsx` | **Dev 2** | 1h | Sprint 5 |
| INFRA-5 | On-Demand revalidation endpoint | `src/app/api/revalidate/route.ts` | **Dev 2** | 2h | Sprint 5 |
| INFRA-6 | E2E tests críticos (Playwright) | `tests/e2e/checkout.spec.ts` | **Dev 2** | 8h | Sprint 8+ |

**INFRA-1 — BFF allowlist (estrutura atual):**

```typescript
// src/app/api/shop/[...path]/route.ts
// Allowlist de endpoints permitidos — expandir conforme stories completadas
const ALLOWED_ENDPOINTS: Record<string, string[]> = {
  "checkout/cart":                       ["GET", "POST", "PUT", "DELETE"],
  "checkout/cart/coupon":                ["POST", "DELETE"],
  "checkout/cart/estimate-shipping-methods": ["POST"],
  "checkout/onepage/addresses":          ["POST"],
  "checkout/onepage/shipping-methods":   ["POST"],
  "checkout/onepage/payment-methods":    ["POST"],
  "checkout/onepage/orders":             ["POST"],
  "customer/wishlist":                   ["GET", "POST", "DELETE"],
  "v1/customer/login":                   ["POST"],
  "v1/customer/register":               ["POST"],
  "v1/customer/logout":                  ["POST"],
  "v1/customer/forgot-password":         ["POST"],
  "v1/customer/get":                     ["GET"],
  "v1/customer/profile":                 ["POST"],
  "v1/customer/addresses":              ["GET", "POST"],
  "v1/customer/orders":                  ["GET"],
  // ... adicionar conforme features implementadas
};
```

**INFRA-5 — On-Demand revalidation (webhooks Bagisto → Next.js):**

```typescript
// src/app/api/revalidate/route.ts
import { NextRequest, NextResponse } from "next/server";
import { revalidatePath, revalidateTag } from "next/cache";

export async function POST(request: NextRequest) {
  const secret = request.headers.get("x-revalidate-secret");
  if (secret !== process.env.REVALIDATE_SECRET) {
    return NextResponse.json({ error: "Unauthorized" }, { status: 401 });
  }

  const { path, tag } = await request.json();
  if (path) revalidatePath(path);
  if (tag) revalidateTag(tag);

  return NextResponse.json({ revalidated: true, timestamp: Date.now() });
}
```

---

## EPIC: AUTH — Autenticação & Conta do Cliente {#auth}

> **Objetivo:** Login, registro, recuperação de senha, perfil, endereços e histórico de pedidos — integrados com iron-session (SEC-2).
> **Responsável:** Dev 2 (toda UI/BFF). Dev 1 colabora em endpoints Bagisto que não existirem.
> **Predecessor:** SEC-2 completo.

| ID | Story | Arquivo | Resp. | Estimativa | Sprint |
|---|---|---|---|---|---|
| AUTH-1 | Página de Login | `src/app/login/page.tsx` | **Dev 2** | 2h | Sprint 1 |
| AUTH-2 | Página de Registro | `src/app/register/page.tsx` | **Dev 2** | 2h | Sprint 1 |
| AUTH-3 | Esqueci senha + Reset | `src/app/esqueci-senha/`, `src/app/redefinir-senha/` | **Dev 2** | 3h | Sprint 3 |
| AUTH-4 | Logout (header/menu) | `src/components/UserMenu.tsx` | **Dev 2** | 1h | Sprint 1 |
| AUTH-5 | Perfil: editar dados + endereços | `src/app/conta/perfil/`, `src/app/conta/enderecos/` | **Dev 2** | 5h | Sprint 3 |
| AUTH-6 | Histórico de pedidos | `src/app/conta/pedidos/`, `src/app/conta/pedidos/[id]/` | **Dev 2** | 4h | Sprint 3 |
| AUTH-7 | Wishlist (opcional pós-launch) | `src/app/conta/wishlist/` | **Dev 2** | 4h | Sprint 8+ |

**AUTH-1 — Login flow:**
1. `POST /api/auth/login` (BFF Route Handler, implementado em SEC-2)
2. Zod parse → `INTERNAL_API_URL` → `POST /api/v1/customer/login`
3. Extrai `laravel_session` cookie da resposta Bagisto
4. Salva em `zhy_session` (iron-session) — browser recebe apenas cookie encriptado
5. Redirect para `?next=` ou `/conta`

**AUTH-5 — Perfil pages:**

```typescript
// src/app/conta/perfil/page.tsx (Server Component)
import { requireAuth } from "@/lib/session";
import ProfileForm from "./_components/ProfileForm";

export default async function PerfilPage() {
  const session = await requireAuth(); // redirect se não autenticado

  const profile = await fetch(
    `${process.env.INTERNAL_API_URL}/api/v1/customer/get`,
    {
      headers: { Cookie: `laravel_session=${session.bagistoSessionCookie}` },
      cache: "no-store",
    }
  ).then(r => r.json());

  return <ProfileForm profile={profile.data} />;
}
```

**AUTH-6 — Pedidos:**

```typescript
// src/app/conta/pedidos/page.tsx (Server Component)
import { requireAuth } from "@/lib/session";

export default async function PedidosPage() {
  const session = await requireAuth();

  const orders = await fetch(
    `${process.env.INTERNAL_API_URL}/api/v1/customer/orders?page=1&limit=10`,
    {
      headers: { Cookie: `laravel_session=${session.bagistoSessionCookie}` },
      cache: "no-store",
    }
  ).then(r => r.json());

  return (/* JSX com tabela de pedidos */null);
}
```

**Acceptance Criteria AUTH:**
- [ ] Login com credenciais inválidas → mensagem de erro clara (sem stack trace)
- [ ] Sessão persiste após reload (cookie 7 dias)
- [ ] `/conta` sem sessão → redirect `/login?next=/conta`
- [ ] Perfil editado reflete no Bagisto Admin

---

## ~~EPIC: PAY~~ — Stripe [DEPRECATED — Veto 2] {#pay}

> **⛔ DECISÃO IRREVOGÁVEL — Veto 2:** Stripe foi eliminado como gateway de pagamento. O código do pacote `packages/Webkul/Stripe/` permanece no repositório apenas para referência histórica do código PIX customizado (`Stripe.php`, `CancelExpiredPixOrders.php`), mas **não deve ser habilitado, referenciado em novas features, nem incluído em qualquer sprint**.

> O gateway exclusivo para o mercado brasileiro é o **Mercado Pago** (`EPIC: PAY-BR`), consolidando cartão de crédito, PIX e boleto bancário em uma única integração, com HMAC-SHA256 no webhook e notificação via SSE.

---

## EPIC: PAY-BR — Pagamentos Brasileiros — Mercado Pago [Veto 2 + Veto 3] {#pay-br}

> **Objetivo:** Integração completa com Mercado Pago: cartão de crédito, PIX (QR Code + SSE) e boleto bancário. Webhook seguro com HMAC-SHA256. PIX com notificação real-time via Server-Sent Events (elimina polling — Veto 3).
> **Responsável:** Dev 1 (Bagisto backend — install, webhook, state machine, jobs). Dev 2 (Next.js — UI/BFF, SSE client, página PIX).

| ID | Story | Resp. | Estimativa | Sprint |
|---|---|---|---|---|
| PAY-BR-1 | Instalar e configurar pacote MP no Bagisto | **Dev 1** | 3h | Sprint 1 |
| PAY-BR-2 | UI de pagamento — boleto bancário | **Dev 2** | 4h | Sprint 2 |
| PAY-BR-3 | UI de pagamento — cartão de crédito | **Dev 2** | 6h | Sprint 2 |
| PAY-BR-4 | Parcelamento sem juros — seleção de parcelas | **Dev 2** | 4h | Sprint 2 |
| PAY-BR-5 | Webhook MP — HMAC-SHA256 + State Machine [Veto 2] | **Dev 1** | 8h | Sprint 2 |
| PAY-BR-6 | PIX SSE — notificação real-time [Veto 3] | **Dev 1** + **Dev 2** | 6h | Sprint 2 |

---

### PAY-BR-1 — Instalar e Configurar Mercado Pago

| Task | Arquivo | Resp. | Estimativa |
|---|---|---|---|
| B17 — Instalar SDK MP | `zhy-core-api/` — `composer require mercadopago/dx-php` | **Dev 1** | 1h |
| B18 — Criar pagamento via MP API | `packages/Webkul/MercadoPago/` ou estender Stripe.php | **Dev 1** | 2h |
| Configurar credenciais | `.env` → `MP_ACCESS_TOKEN`, `MP_WEBHOOK_SECRET` | **Dev 1** | 30 min |

---

### PAY-BR-5 — Webhook MP: HMAC-SHA256 + State Machine [Veto 2]

> **Rationale Veto 2:** HMAC-SHA256 no webhook impede que terceiros simulem confirmações de pagamento falsas. O formato de assinatura do MP é específico: `ts=TIMESTAMP,v1=HMAC_SHA256(secret, "id:{id};request-id:{x-request-id};ts:{ts}")`.

**Arquivo:** `zhy-core-api/packages/Webkul/MercadoPago/Http/Controllers/MercadoPagoWebhookController.php`

```php
<?php

namespace Webkul\MercadoPago\Http\Controllers;

use Illuminate\Http\Request;
use Illuminate\Support\Facades\Log;
use Illuminate\Routing\Controller;
use Webkul\MercadoPago\Jobs\ProcessMercadoPagoPayment;

class MercadoPagoWebhookController extends Controller
{
    public function handle(Request $request): \Illuminate\Http\JsonResponse
    {
        // HMAC-SHA256 validation — Veto 2 obrigatório
        $xSignature   = $request->header('x-signature', '');
        $xRequestId   = $request->header('x-request-id', '');
        $dataId       = $request->input('data.id', '');
        $secret       = config('services.mercadopago.webhook_secret');

        // Parse ts e v1 do header
        $parts = [];
        foreach (explode(',', $xSignature) as $part) {
            [$k, $v] = array_pad(explode('=', $part, 2), 2, '');
            $parts[trim($k)] = trim($v);
        }

        $ts = $parts['ts'] ?? '';
        $v1 = $parts['v1'] ?? '';

        if (empty($ts) || empty($v1)) {
            Log::warning('MP Webhook: header x-signature malformado');
            return response()->json(['error' => 'Invalid signature format'], 400);
        }

        // Proteger contra replay attacks (30 segundos de tolerância)
        if (abs(time() - (int) $ts) > 30) {
            Log::warning('MP Webhook: timestamp expirado', ['ts' => $ts]);
            return response()->json(['error' => 'Timestamp expired'], 400);
        }

        $manifest = "id:{$dataId};request-id:{$xRequestId};ts:{$ts}";
        $computed = hash_hmac('sha256', $manifest, $secret);

        if (!hash_equals($computed, $v1)) {
            Log::warning('MP Webhook: assinatura inválida', ['data_id' => $dataId]);
            return response()->json(['error' => 'Invalid signature'], 401);
        }

        $type   = $request->input('type');
        $action = $request->input('action');

        // Despachar para fila Redis (não bloquear o webhook)
        ProcessMercadoPagoPayment::dispatch($dataId, $type, $action)
            ->onQueue('payments');

        return response()->json(['status' => 'queued']);
    }
}
```

**Job: ProcessMercadoPagoPayment — State Machine:**

```php
<?php

namespace Webkul\MercadoPago\Jobs;

use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Support\Facades\Redis;
use Illuminate\Support\Facades\Log;

class ProcessMercadoPagoPayment implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable;

    public int $tries = 3;
    public int $backoff = 30; // segundos entre retentativas

    private array $validTransitions = [
        'pending'    => ['approved', 'rejected', 'cancelled', 'in_process'],
        'in_process' => ['approved', 'rejected', 'cancelled'],
        'approved'   => [],  // estado final
        'rejected'   => [],  // estado final
        'cancelled'  => [],  // estado final
    ];

    public function __construct(
        private string $paymentId,
        private string $type,
        private string $action
    ) {}

    public function handle(): void
    {
        if ($this->type !== 'payment') {
            Log::info('MP Webhook: tipo não-payment ignorado', ['type' => $this->type]);
            return;
        }

        // Buscar detalhes do pagamento na API MP
        $payment = $this->fetchPaymentFromMP($this->paymentId);
        if (!$payment) {
            Log::error('MP: pagamento não encontrado', ['id' => $this->paymentId]);
            return;
        }

        $externalOrderId = $payment['external_reference'];
        $newStatus       = $payment['status'];

        $order = \Webkul\Sales\Models\Order::find($externalOrderId);
        if (!$order) {
            Log::error('MP: pedido não encontrado', ['order_id' => $externalOrderId]);
            return;
        }

        $currentStatus = $order->status;

        // Validação da state machine
        if (!in_array($newStatus, $this->validTransitions[$currentStatus] ?? [])) {
            Log::warning('MP: transição inválida na state machine', [
                'from' => $currentStatus,
                'to'   => $newStatus,
                'order_id' => $externalOrderId,
            ]);
            return;
        }

        // Atualizar status do pedido
        if ($newStatus === 'approved') {
            $order->update(['status' => 'processing']);

            // Publicar confirmação para SSE (Veto 3)
            Redis::publish(
                "pix:confirmed:{$externalOrderId}",
                json_encode([
                    'orderId'   => $externalOrderId,
                    'status'    => 'approved',
                    'paymentId' => $this->paymentId,
                    'timestamp' => now()->toISOString(),
                ])
            );

            Log::info('MP: PIX confirmado e publicado no Redis', ['order_id' => $externalOrderId]);
        } elseif (in_array($newStatus, ['rejected', 'cancelled'])) {
            $order->update(['status' => 'canceled']);
            Log::info('MP: pagamento rejeitado/cancelado', ['order_id' => $externalOrderId, 'status' => $newStatus]);
        }
    }

    private function fetchPaymentFromMP(string $paymentId): ?array
    {
        // Implementação via SDK mercadopago/dx-php ou Guzzle
        // Retorna array com status, external_reference, etc.
        return null; // placeholder — implementar com SDK
    }
}
```

---

### PAY-BR-6 — PIX SSE: Server-Sent Events + Redis Pub/Sub [Veto 3]

> **Rationale Veto 3:** Polling a cada 10s com 100 usuários simultâneos = 18.000 req/h desnecessários. SSE permite que o servidor notifique o cliente exatamente uma vez na confirmação, via Redis Pub/Sub. Um heartbeat de 25s mantém a conexão viva através de proxies e load balancers.

**Arquivo Backend:** `src/app/api/pix/sse/[orderId]/route.ts`

```typescript
import { NextRequest } from "next/server";
import Redis from "ioredis";

export const runtime = "nodejs";
export const dynamic = "force-dynamic";

export async function GET(
  request: NextRequest,
  { params }: { params: { orderId: string } }
) {
  const { orderId } = params;

  const encoder = new TextEncoder();
  let subscriber: Redis | null = null;

  const stream = new ReadableStream({
    async start(controller) {
      subscriber = new Redis(process.env.REDIS_URL!);

      const channel = `pix:confirmed:${orderId}`;

      // Heartbeat a cada 25s para manter conexão viva (proxies costumam fechar após 30s)
      const heartbeat = setInterval(() => {
        try {
          controller.enqueue(encoder.encode(": heartbeat\n\n"));
        } catch {
          clearInterval(heartbeat);
        }
      }, 25_000);

      await subscriber.subscribe(channel);

      subscriber.on("message", (ch, message) => {
        if (ch !== channel) return;
        controller.enqueue(encoder.encode(`event: pix_confirmed\ndata: ${message}\n\n`));
        cleanup();
      });

      const cleanup = () => {
        clearInterval(heartbeat);
        subscriber?.unsubscribe(channel);
        subscriber?.quit();
        try { controller.close(); } catch { /* já fechado */ }
      };

      // Abort quando cliente desconecta
      request.signal.addEventListener("abort", cleanup);
    },
  });

  return new Response(stream, {
    headers: {
      "Content-Type": "text/event-stream",
      "Cache-Control": "no-cache, no-transform",
      Connection: "keep-alive",
      "X-Accel-Buffering": "no", // desativa buffering do Nginx
    },
  });
}
```

**Arquivo Frontend:** `src/app/pedido/pix/[orderId]/page.tsx`

```tsx
"use client";

import { useEffect, useState } from "react";
import { useRouter } from "next/navigation";
import Image from "next/image";

interface PixPageProps {
  params: { orderId: string };
  qrCodeBase64: string;
  pixKey: string;
}

export default function PixWaitingPage({
  params: { orderId },
  qrCodeBase64,
  pixKey,
}: PixPageProps) {
  const router = useRouter();
  const [status, setStatus] = useState<"waiting" | "confirmed" | "expired">("waiting");

  useEffect(() => {
    const eventSource = new EventSource(`/api/pix/sse/${orderId}`);

    eventSource.addEventListener("pix_confirmed", () => {
      setStatus("confirmed");
      eventSource.close();
      // Redirecionar para página de sucesso após 2s
      setTimeout(() => router.push(`/pedido/sucesso/${orderId}`), 2_000);
    });

    eventSource.onerror = () => {
      // Reconectar automaticamente — EventSource faz isso por padrão
      console.warn("SSE: conexão interrompida, reconectando...");
    };

    // Expirar após 30 minutos (PIX expira no Bagisto/MP)
    const expiry = setTimeout(() => {
      setStatus("expired");
      eventSource.close();
    }, 30 * 60 * 1_000);

    return () => {
      eventSource.close();
      clearTimeout(expiry);
    };
  }, [orderId, router]);

  return (
    <div className="min-h-screen bg-[#F5EFE5] flex flex-col items-center justify-center p-8">
      {status === "waiting" && (
        <>
          <h1 className="font-title text-2xl text-[#1F2430] mb-4">
            Aguardando pagamento PIX
          </h1>
          <Image
            src={`data:image/png;base64,${qrCodeBase64}`}
            alt="QR Code PIX"
            width={256}
            height={256}
            className="rounded-lg shadow-lg"
          />
          <p className="mt-4 text-sm text-[#8B7355]">
            Copie a chave PIX ou escaneie o QR Code
          </p>
          <code className="mt-2 bg-white px-4 py-2 rounded text-sm break-all max-w-xs text-center">
            {pixKey}
          </code>
          <div className="mt-6 flex items-center gap-2 text-[#8B7355]">
            <div className="animate-spin h-4 w-4 border-2 border-[#8B7355] border-t-transparent rounded-full" />
            <span className="text-sm">Aguardando confirmação...</span>
          </div>
        </>
      )}

      {status === "confirmed" && (
        <div className="text-center">
          <div className="text-5xl mb-4">✅</div>
          <h1 className="font-title text-2xl text-[#1F2430]">PIX confirmado!</h1>
          <p className="text-[#8B7355] mt-2">Redirecionando...</p>
        </div>
      )}

      {status === "expired" && (
        <div className="text-center">
          <h1 className="font-title text-2xl text-[#1F2430]">PIX expirado</h1>
          <p className="text-[#8B7355] mt-2">O QR Code expirou. Gere um novo pedido.</p>
          <a href="/carrinho" className="mt-4 inline-block bg-[#8B7355] text-white px-6 py-3 rounded">
            Voltar ao carrinho
          </a>
        </div>
      )}
    </div>
  );
}
```

**Acceptance Criteria PAY-BR-6:**
- [ ] Webhook MP recebido → Redis `PUBLISH pix:confirmed:{orderId}` → SSE entrega evento ao browser em < 1s
- [ ] Browser redireciona para `/pedido/sucesso/{id}` após confirmação SSE
- [ ] Heartbeat enviado a cada 25s (verificar com `curl -N /api/pix/sse/TEST-123`)
- [ ] Conexão SSE abortada pelo client → subscriber Redis desconecta (sem leak)
- [ ] PIX expirado após 30 min → status "expired" no UI

---
## EPIC: SHIP — Frete & Logística Brasil {#ship}

> **Objetivo:** Integrar com transportadoras brasileiras para cotação e seleção de frete no checkout.
> **Responsável:** Dev 1 (Bagisto config + carrier setup). Dev 2 (UI frete no checkout).

| ID | Story | Arquivo | Resp. | Estimativa | Sprint |
|---|---|---|---|---|---|
| SHIP-1 | Instalar + configurar Melhor Envio | `packages/Webkul/MelhorEnvio/` ou integração via Bagisto Shipping | **Dev 1** | 5h | Sprint 3 |
| SHIP-2 | Cotação automática por CEP | Endpoint `POST /api/checkout/cart/estimate-shipping-methods` + UI | **Dev 1** | 3h | Sprint 3 |
| SHIP-3 | UI de seleção de frete no checkout | `src/app/checkout/` — componente ShippingMethodSelector | **Dev 2** | 3h | Sprint 3 |
| SHIP-4 | Frete grátis: regra de valor mínimo (R$ X) | Config Bagisto Admin + lógica no carrinho | **Dev 1** | 2h | Sprint 8+ |
| SHIP-5 | Bloqueio B19: configurar carrier padrão | `zhy-core-api/config/` ou Bagisto Admin | **Dev 1** | 2h | Sprint 3 |

**SHIP-3 — Componente de seleção de frete:**

```typescript
// src/app/checkout/_components/ShippingMethodSelector.tsx
"use client";

interface ShippingMethod {
  code: string;
  title: string;
  price: number;
  carrier: string;
  delivery_days?: number;
}

export default function ShippingMethodSelector({
  methods,
  selected,
  onSelect,
}: {
  methods: ShippingMethod[];
  selected: string;
  onSelect: (code: string) => void;
}) {
  return (
    <div className="space-y-3">
      {methods.map((method) => (
        <label
          key={method.code}
          className={`flex items-center justify-between p-4 rounded-lg border cursor-pointer
            ${selected === method.code
              ? "border-[#8B7355] bg-[#F5EFE5]"
              : "border-gray-200 hover:border-[#8B7355]/50"
            }`}
        >
          <input
            type="radio"
            name="shipping"
            value={method.code}
            checked={selected === method.code}
            onChange={() => onSelect(method.code)}
            className="sr-only"
          />
          <div>
            <p className="font-medium text-[#1F2430]">{method.title}</p>
            {method.delivery_days && (
              <p className="text-sm text-[#8B7355]">
                Entrega em até {method.delivery_days} dias úteis
              </p>
            )}
          </div>
          <span className="font-semibold text-[#1F2430]">
            {method.price === 0
              ? "Grátis"
              : `R$ ${method.price.toFixed(2)}`}
          </span>
        </label>
      ))}
    </div>
  );
}
```

**Bloqueadores SHIP:**
- **B19:** Carrier Melhor Envio não configurado no Bagisto — cotação retorna vazia
- **B20:** Endereço de origem do vendedor não cadastrado no Bagisto — cotação incorreta

**Acceptance Criteria SHIP:**
- [ ] `POST /api/checkout/cart/estimate-shipping-methods` com CEP válido retorna ≥ 1 opção de frete
- [ ] Frete grátis ativo acima de R$ X (valor a definir — ver Pendências)
- [ ] Seleção de frete refletida no total do checkout

---

## EPIC: CAT — Catálogo & Navegação {#cat}

> **Objetivo:** Navegação por categorias, PLP com filtros e busca, performance via ISR.
> **Responsável:** Dev 2 (toda UI Next.js). Dev 1 não tem tarefas neste EPIC (configuração de catálogo é via Bagisto Admin).

| ID | Story | Arquivo | Resp. | Estimativa | Sprint |
|---|---|---|---|---|---|
| CAT-1 | Menu de navegação por categorias | `src/components/CategoryNav.tsx` | **Dev 2** | 3h | Sprint 5 |
| CAT-2 | PLP — filtros (preço, atributos) + paginação | `src/app/produtos/page.tsx` — filtros client-side | **Dev 2** | 6h | Sprint 5 |
| CAT-3 | Busca de produtos (client-side ou Bagisto search) | `src/app/busca/page.tsx` | **Dev 2** | 4h | Sprint 5 |
| CAT-4 | Breadcrumbs dinâmicos | `src/components/Breadcrumbs.tsx` | **Dev 2** | 2h | Sprint 5 |

**CAT-2 — PLP com filtros:**

```typescript
// src/app/produtos/page.tsx (Server Component com ISR)
import { Suspense } from "react";
import ProductGrid from "./_components/ProductGrid";
import FilterSidebar from "./_components/FilterSidebar";

export const revalidate = 300; // ISR: revalidar a cada 5 minutos

interface ProductsPageProps {
  searchParams: {
    category_id?: string;
    sort?: string;
    order?: "asc" | "desc";
    page?: string;
    min_price?: string;
    max_price?: string;
  };
}

export default async function ProductsPage({ searchParams }: ProductsPageProps) {
  const params = new URLSearchParams(searchParams as Record<string, string>);
  params.set("limit", "24");

  const [products, categories] = await Promise.all([
    fetch(`${process.env.INTERNAL_API_URL}/api/v1/products?${params}`).then(r => r.json()),
    fetch(`${process.env.INTERNAL_API_URL}/api/v1/categories`).then(r => r.json()),
  ]);

  return (
    <div className="flex gap-8 max-w-7xl mx-auto px-4 py-8">
      <FilterSidebar
        categories={categories.data}
        currentParams={searchParams}
      />
      <Suspense fallback={<div>Carregando produtos...</div>}>
        <ProductGrid products={products.data} meta={products.meta} />
      </Suspense>
    </div>
  );
}
```

**CAT-1 — Menu de categorias:**

```typescript
// src/components/CategoryNav.tsx (Server Component)
export default async function CategoryNav() {
  const res = await fetch(
    `${process.env.INTERNAL_API_URL}/api/categories/tree`,
    { next: { revalidate: 3600 } } // 1 hora de cache
  );
  const { data } = await res.json();

  return (
    <nav className="flex gap-6 items-center">
      {data?.map((cat: { id: number; name: string; slug: string }) => (
        <a
          key={cat.id}
          href={`/produtos?category_id=${cat.id}`}
          className="text-sm font-medium text-[#1F2430] hover:text-[#8B7355] transition-colors"
        >
          {cat.name}
        </a>
      ))}
    </nav>
  );
}
```

**Acceptance Criteria CAT:**
- [ ] Menu exibe categorias reais do Bagisto (não hardcoded)
- [ ] Filtro por preço funciona com `min_price` / `max_price` query params
- [ ] Busca retorna resultados relevantes
- [ ] ISR revalidada quando produto novo é adicionado (on-demand via INFRA-5)

---

## EPIC: PROD — Produto Detalhe & Avaliações {#prod}

> **Objetivo:** PDP com variantes, galeria de imagens, avaliações de clientes.
> **Responsável:** Dev 2 (toda UI). Dev 1 não tem tarefas diretas.

| ID | Story | Arquivo | Resp. | Estimativa | Sprint |
|---|---|---|---|---|---|
| PROD-1 | PDP — produto simples com galeria | `src/app/produtos/[slug]/page.tsx` | **Dev 2** | 4h | Sprint 5 |
| PROD-2 | PDP — produto configurável (variantes) | `src/app/produtos/[slug]/page.tsx` + `VariantSelector.tsx` | **Dev 2** | 6h | Sprint 5 |
| PROD-3 | Avaliações — listar + formulário | `src/app/produtos/[slug]/_components/Reviews.tsx` | **Dev 2** | 5h | Sprint 8+ |
| PROD-4 | Produtos relacionados / "Você pode gostar" | `src/app/produtos/[slug]/_components/RelatedProducts.tsx` | **Dev 2** | 3h | Sprint 8+ |

**PROD-2 — Seletor de variantes:**

```typescript
// src/app/produtos/[slug]/_components/VariantSelector.tsx
"use client";

interface SuperAttribute {
  id: number;
  code: string;
  label: string;
  options: { id: string; label: string; isOutOfStock: boolean }[];
}

export default function VariantSelector({
  superAttributes,
  onVariantChange,
}: {
  superAttributes: SuperAttribute[];
  onVariantChange: (attrs: Record<string, string>) => void;
}) {
  const [selected, setSelected] = useState<Record<string, string>>({});

  const handleSelect = (attributeId: number, optionId: string) => {
    const next = { ...selected, [attributeId]: optionId };
    setSelected(next);
    if (Object.keys(next).length === superAttributes.length) {
      onVariantChange(next);
    }
  };

  return (
    <div className="space-y-4">
      {superAttributes.map((attr) => (
        <div key={attr.id}>
          <p className="text-sm font-medium text-[#1F2430] mb-2">{attr.label}</p>
          <div className="flex flex-wrap gap-2">
            {attr.options.map((opt) => (
              <button
                key={opt.id}
                onClick={() => handleSelect(attr.id, opt.id)}
                disabled={opt.isOutOfStock}
                className={`px-4 py-2 rounded-lg text-sm border transition-colors
                  ${selected[attr.id] === opt.id
                    ? "bg-[#8B7355] text-white border-[#8B7355]"
                    : opt.isOutOfStock
                    ? "opacity-40 cursor-not-allowed bg-gray-100"
                    : "border-gray-300 hover:border-[#8B7355]"
                  }`}
              >
                {opt.label}
              </button>
            ))}
          </div>
        </div>
      ))}
    </div>
  );
}
```

**Acceptance Criteria PROD:**
- [ ] PDP carrega com ISR (revalidate 5 min)
- [ ] Seleção de variante atualiza preço e disponibilidade em tempo real
- [ ] "Adicionar ao carrinho" desabilitado até todas as variantes serem selecionadas

---

## EPIC: CART — Carrinho & Checkout {#cart}

> **Objetivo:** Carrinho persistente, checkout multi-step completo, página de sucesso.
> **Responsável:** Dev 2 (toda UI/BFF checkout). Dev 1 (configurações Bagisto para checkout funcionar: B17, B18, B19, B20, B21).

| ID | Story | Arquivo | Resp. | Estimativa | Sprint |
|---|---|---|---|---|---|
| CART-1 | Carrinho — página + atualização de quantidade | `src/app/carrinho/`, `CartContext.tsx` | **Dev 2** | 4h | Sprint 6 |
| CART-2 | Cupom de desconto | `src/app/carrinho/_components/CouponInput.tsx` | **Dev 2** | 3h | Sprint 8+ |
| CART-3 | Checkout multi-step: endereço + frete + pagamento | `src/app/checkout/page.tsx` | **Dev 2** | 10h | Sprint 6 |
| CART-4 | Página de sucesso do pedido | `src/app/pedido/sucesso/[id]/page.tsx` | **Dev 2** | 3h | Sprint 6 |

**CART-3 — Estrutura do checkout multi-step:**

```typescript
// src/app/checkout/page.tsx
"use client";

import { useState } from "react";
import AddressStep from "./_components/AddressStep";
import ShippingStep from "./_components/ShippingStep";
import PaymentStep from "./_components/PaymentStep";
import ReviewStep from "./_components/ReviewStep";

type Step = "address" | "shipping" | "payment" | "review";

export default function CheckoutPage() {
  const [step, setStep] = useState<Step>("address");
  const [checkoutData, setCheckoutData] = useState({});

  const steps: Step[] = ["address", "shipping", "payment", "review"];

  return (
    <div className="min-h-screen bg-[#F5EFE5]">
      {/* Progress indicator */}
      <div className="flex justify-center gap-4 py-6">
        {steps.map((s, i) => (
          <div key={s} className={`flex items-center gap-2
            ${steps.indexOf(step) >= i ? "text-[#8B7355]" : "text-gray-400"}`}>
            <span className={`w-7 h-7 rounded-full flex items-center justify-center text-sm font-bold
              ${steps.indexOf(step) >= i ? "bg-[#8B7355] text-white" : "bg-gray-200"}`}>
              {i + 1}
            </span>
            <span className="text-sm capitalize hidden sm:inline">{s}</span>
          </div>
        ))}
      </div>

      {/* Steps */}
      {step === "address"  && <AddressStep  onNext={(d) => { setCheckoutData(p => ({...p, ...d})); setStep("shipping"); }} />}
      {step === "shipping" && <ShippingStep onNext={(d) => { setCheckoutData(p => ({...p, ...d})); setStep("payment"); }} />}
      {step === "payment"  && <PaymentStep  onNext={(d) => { setCheckoutData(p => ({...p, ...d})); setStep("review"); }} />}
      {step === "review"   && <ReviewStep   data={checkoutData} />}
    </div>
  );
}
```

**CART-2 — Cupom de desconto:**

```typescript
// src/app/carrinho/_components/CouponInput.tsx
"use client";

export default function CouponInput({ onApply }: { onApply: (discount: number) => void }) {
  const [code, setCode] = useState("");
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  const handleApply = async () => {
    setLoading(true);
    setError(null);
    try {
      const res = await fetch("/api/shop/checkout/cart/coupon", {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
          "x-csrf-token": document.cookie.match(/csrf_token=([^;]+)/)?.[1] ?? "",
        },
        body: JSON.stringify({ code }),
      });

      if (!res.ok) {
        const data = await res.json();
        setError(data.message ?? "Cupom inválido");
        return;
      }

      const cart = await res.json();
      onApply(cart.data?.discount_amount ?? 0);
    } finally {
      setLoading(false);
    }
  };

  return (
    <div className="flex gap-2">
      <input
        type="text"
        value={code}
        onChange={(e) => setCode(e.target.value.toUpperCase())}
        placeholder="CÓDIGO DO CUPOM"
        className="flex-1 border rounded px-3 py-2 text-sm"
      />
      <button
        onClick={handleApply}
        disabled={loading || !code}
        className="bg-[#8B7355] text-white px-4 py-2 rounded text-sm disabled:opacity-50"
      >
        {loading ? "..." : "Aplicar"}
      </button>
      {error && <p className="text-red-500 text-xs mt-1 col-span-2">{error}</p>}
    </div>
  );
}
```

**Acceptance Criteria CART:**
- [ ] Adicionar/remover/atualizar quantidade no carrinho — estado persistido no Bagisto (não só localStorage)
- [ ] Checkout completo: endereço → frete → pagamento → pedido criado no Bagisto
- [ ] Página de sucesso mostra número do pedido, itens e total
- [ ] Cupom de desconto: aplicado com sucesso → desconto exibido no resumo

---
## Planejamento de Sprints {#sprints}

> **Metodologia:** Scrum. Cada sprint = **1 semana**. Velocidade estimada: Dev 1 ≈ 25–30h/semana. Dev 2 ≈ 25–30h/semana.
> **Legenda:** SP = Story Points (1 SP ≈ 1h). Dev 1 = Backend/Infra. Dev 2 = Frontend/BFF.
> **Definition of Done:** Código em `main`, CI verde, testado em staging, sem erros em `laravel.log` nem no console do browser.

---

### Sprint 0 — Semana 1: Segurança Crítica + Verificação de Staging {#sprint-0}

> **Objetivo:** Corrigir as 4 vulnerabilidades críticas imediatas (SEC-1) + verificar o estado real da codebase em staging (V1–V14) + resolver os bloqueadores de infra básicos.

**Dev 1 — Backend/Infra:**

| Story / Task | SP | Repo | Notas |
|---|---|---|---|
| SEC-1.1 — Mailpit para 127.0.0.1 | 0.5 | `zhy-infra` | V-SEC-6 resolvido |
| SEC-1.2 — Sanctum expiration 7 dias | 0.5 | `zhy-core-api` | V-SEC-4 resolvido |
| SEC-1.3 — Session secure cookie | 0.5 | `zhy-core-api` | V-SEC-5 resolvido |
| B7 — `QUEUE_CONNECTION=redis` em staging | 0.5 | `zhy-core-api` | V13 depende disso |
| B8 — Agendar `CancelExpiredPixOrders` | 0.5 | `zhy-core-api` | `App\Console\Kernel` |
| B6 — `Dockerfile.prod` inicial (PHP-FPM) | 3 | `zhy-core-api` | Substituir `artisan serve` |
| Subir ambiente de staging | 5 | `zhy-infra` | `docker-compose.yml` com `.env.staging` |
| V13 — Verificar PIX em staging | 5 | `zhy-core-api` | Job rodando, webhook acessível |
| V14 — Executar `php artisan test` | 2 | `zhy-core-api` | `--filter=Checkout` — 0 falhas |
| **Total Dev 1** | **17.5 SP** | | |

**Dev 2 — Frontend/BFF:**

| Story / Task | SP | Repo | Notas |
|---|---|---|---|
| SEC-1.4 — `remotePatterns` no `next.config.ts` | 0.5 | `zhy-storefront-crm` | V-SEC-11 resolvido |
| V1 — Homepage com produtos reais em staging | 1 | `zhy-storefront-crm` | `FeaturedProducts` com dados reais |
| V2 — PLP: paginação + ordenação em staging | 1 | `zhy-storefront-crm` | Verificar ISR |
| V3 — PDP: `<Image>` + variantes em staging | 1 | `zhy-storefront-crm` | Após B13 (SEC-1.4) |
| V4 — Carrinho: add/remove/update em staging | 1 | `zhy-storefront-crm` | Persistência no Bagisto |
| V5 — Checkout flow em staging (sem MP ainda) | 2 | `zhy-storefront-crm` | Verificar steps 1–4 |
| V6 — CEP ViaCEP em staging | 1 | `zhy-storefront-crm` | CEP válido + inválido |
| V7 — CPF/CNPJ validação em staging | 0.5 | `zhy-storefront-crm` | Dígitos verificadores |
| V8 — Página de sucesso em staging | 1 | `zhy-storefront-crm` | Pedido real |
| V9 — BFF allowlist smoke test | 1 | `zhy-storefront-crm` | Path fora da allowlist → 404 |
| V10 — Sitemap XML em staging | 0.5 | `zhy-storefront-crm` | `/sitemap.xml` com URLs reais |
| V11 — Nginx headers em staging | 0.5 | `zhy-infra` | `curl -I` em staging |
| V12 — CI zhy-infra funciona | 0.5 | `zhy-infra` | Push → CI verde |
| **Total Dev 2** | **10.5 SP** | | |

**Critério de saída Sprint 0:** SEC-1 ✅ | V1–V12 verificados ✅ | V13 verificado ✅ | V14 zero falhas ✅

---

### Sprint 1 — Semana 2: Iron Session Auth + Mercado Pago Base {#sprint-1}

> **Objetivo:** Implementar auth stateless com iron-session (SEC-2) + instalar SDK do Mercado Pago (PAY-BR-1) + login/logout UI funcional.

**Dev 1 — Backend/Infra:**

| Story / Task | SP | Repo | Notas |
|---|---|---|---|
| PAY-BR-1 — `composer require mercadopago/dx-php` | 1 | `zhy-core-api` | B17 resolvido |
| PAY-BR-1 — Configurar credenciais MP no `.env` | 0.5 | `zhy-core-api` | `MP_ACCESS_TOKEN`, `MP_WEBHOOK_SECRET` |
| PAY-BR-1 — Criar pagamento PIX via MP API | 3 | `zhy-core-api` | B18 resolvido |
| INFRA-2 — `shopClient.ts` retry + backoff | 2 | `zhy-storefront-crm` | Review da implementação |
| **Total Dev 1** | **6.5 SP** | | |

**Dev 2 — Frontend/BFF:**

| Story / Task | SP | Repo | Notas |
|---|---|---|---|
| SEC-2.1 — `src/lib/session.ts` | 1 | `zhy-storefront-crm` | `SessionData` + `getSession()` + `requireAuth()` |
| SEC-2.2 — Route Handler login | 2 | `zhy-storefront-crm` | Zod parse, BFF → Bagisto, iron-session save |
| SEC-2.3 — Route Handler me | 0.5 | `zhy-storefront-crm` | GET sessão |
| SEC-2.4 — Route Handler logout | 0.5 | `zhy-storefront-crm` | `session.destroy()` |
| SEC-2.5 — `src/middleware.ts` | 1 | `zhy-storefront-crm` | Proteger `/conta`, `/checkout`, `/pedido` |
| AUTH-1 — Página Login (UI ZHY) | 2 | `zhy-storefront-crm` | Glassmorphism, `font-title` Caslon |
| AUTH-2 — Página Registro (UI ZHY) | 2 | `zhy-storefront-crm` | Form validado com Zod |
| AUTH-4 — Logout no menu/header | 1 | `zhy-storefront-crm` | `UserMenu.tsx` |
| **Total Dev 2** | **10 SP** | | |

**Critério de saída Sprint 1:** Login funcional ✅ | `zhy_session` cookie httpOnly ✅ | `/conta` protegida ✅ | SDK MP instalado ✅

---

### Sprint 2 — Semana 3: BFF Hardening + Pagamentos BR Completo {#sprint-2}

> **Objetivo:** Completar SEC-3 (CSRF + rate limit + Zod) + implementar toda a suite PAY-BR-2/3/4/5/6.

**Dev 1 — Backend/Infra:**

| Story / Task | SP | Repo | Notas |
|---|---|---|---|
| PAY-BR-5 — `MercadoPagoWebhookController` (HMAC-SHA256) | 4 | `zhy-core-api` | Veto 2 — validação de assinatura obrigatória |
| PAY-BR-5 — `ProcessMercadoPagoPayment` Job (State Machine) | 4 | `zhy-core-api` | `validTransitions[]` + `Redis::publish()` |
| PAY-BR-5 — Route + Middleware webhook no Bagisto | 1 | `zhy-core-api` | `POST /api/webhooks/mercadopago` |
| B9 — Supervisor rodando queue:work | 2 | `zhy-core-api` | Substituir `queue:work` manual |
| **Total Dev 1** | **11 SP** | | |

**Dev 2 — Frontend/BFF:**

| Story / Task | SP | Repo | Notas |
|---|---|---|---|
| SEC-3.1 — CSRF Route Handler + validação BFF | 3 | `zhy-storefront-crm` | Veto 2 segurança |
| SEC-3.2 — Rate limiter (`authRateLimiter`, `bffRateLimiter`, `ordersRateLimiter`) | 2 | `zhy-storefront-crm` | 5/15min, 100/min, 5/hr |
| SEC-3.3 — Zod schemas (`AddressSchema`, `CartItemSchema`, `CouponSchema`) | 3 | `zhy-storefront-crm` | V-SEC-3 e V-SEC-7 |
| SEC-3.4 — `src/lib/env.ts` (`@t3-oss/env-nextjs`) | 1 | `zhy-storefront-crm` | V-SEC-12 resolvido |
| SEC-3.5 — `.env.example` completo | 0.5 | `zhy-storefront-crm` | B14 resolvido |
| PAY-BR-2 — UI boleto bancário | 4 | `zhy-storefront-crm` | Step de pagamento no checkout |
| PAY-BR-3 — UI cartão de crédito | 6 | `zhy-storefront-crm` | MP Brick ou form custom |
| PAY-BR-4 — Parcelamento: seleção de parcelas | 4 | `zhy-storefront-crm` | Exibir juros por parcela |
| PAY-BR-6 — SSE Route Handler (`/api/pix/sse/[orderId]`) | 3 | `zhy-storefront-crm` | Veto 3 — ioredis subscriber |
| PAY-BR-6 — Página PIX waiting (`/pedido/pix/[orderId]`) | 3 | `zhy-storefront-crm` | `EventSource`, confirmação auto-redirect |
| **Total Dev 2** | **29.5 SP** | | |

> ⚠️ Sprint 2 tem capacidade alta para Dev 2. Ajustar com Product Owner se necessário — mover PAY-BR-3/4 para Sprint 3 caso Sprint 2 travasse.

**Critério de saída Sprint 2:** CSRF ativo ✅ | Rate limit 429 em brute-force ✅ | PIX SSE notifica browser ✅ | Boleto funcional ✅

---

### Sprint 3 — Semana 4: Auth Completo + Frete {#sprint-3}

> **Objetivo:** Completar fluxo de conta do cliente (AUTH-3/5/6) + integrar frete real (SHIP-1/2/3/5) + configurações Bagisto de shipping.

**Dev 1 — Backend/Infra:**

| Story / Task | SP | Repo | Notas |
|---|---|---|---|
| SHIP-1 — Instalar + configurar Melhor Envio | 5 | `zhy-core-api` | B19 resolvido |
| SHIP-2 — Cotação por CEP funcionando | 3 | `zhy-core-api` | `estimate-shipping-methods` retorna ≥ 1 opção |
| SHIP-5 — Carrier padrão + endereço de origem | 2 | `zhy-core-api` | B20 resolvido |
| B21 — SMTP de produção configurado | 2 | `zhy-core-api` | Email de recuperação de senha funcional |
| INFRA-1 — Extend BFF allowlist com novos endpoints | 1 | `zhy-storefront-crm` | Revisar e aprovar expansões |
| **Total Dev 1** | **13 SP** | | |

**Dev 2 — Frontend/BFF:**

| Story / Task | SP | Repo | Notas |
|---|---|---|---|
| AUTH-3 — Esqueci senha + reset | 3 | `zhy-storefront-crm` | Email via B21 (SMTP) |
| AUTH-5 — Perfil: editar dados + endereços | 5 | `zhy-storefront-crm` | `requireAuth()` + Server Component |
| AUTH-6 — Histórico de pedidos | 4 | `zhy-storefront-crm` | Lista + detalhe do pedido |
| SHIP-3 — UI seleção de frete no checkout | 3 | `zhy-storefront-crm` | `ShippingMethodSelector.tsx` |
| **Total Dev 2** | **15 SP** | | |

**Critério de saída Sprint 3:** Recuperação de senha funcional ✅ | Perfil editável ✅ | Frete cotado por CEP ✅ | Shipping no checkout ✅

---

### Sprint 4 — Semana 5: CI/CD + Docker Produção + Disaster Recovery {#sprint-4}

> **Objetivo:** Infraestrutura enterprise completa — imagens imutáveis via GHCR (Veto 1), Docker Compose prod, deploy script, backup automatizado para S3 (Veto 4).

**Dev 1 — Backend/Infra:**

| Story / Task | SP | Repo | Notas |
|---|---|---|---|
| OPS-1.1 — `docker-compose.prod.yml` | 4 | `zhy-infra` | B1 resolvido |
| OPS-1.2 — `supervisor.conf` | 2 | `zhy-core-api` | B9 resolvido (prod) |
| OPS-1.3 — `Dockerfile.prod` Bagisto | 3 | `zhy-core-api` | B6 resolvido |
| OPS-1.5 — `nginx/conf.d/prod.conf` (HTTPS + rate limit) | 3 | `zhy-infra` | B2 resolvido |
| OPS-1.6 — Certbot Let's Encrypt no VPS | 1 | VPS | B3 resolvido |
| OPS-3.2 — CI core-api (PHPUnit + Pint) | 3 | `zhy-core-api` | V12 expandido |
| OPS-3.3 — CD core-api → GHCR | 2 | `zhy-core-api` | Veto 1 |
| OPS-3.5 — `deploy.sh` | 2 | `zhy-infra` | B16 resolvido |
| OPS-4.1–4.5 — MySQL backup + cron + S3 + IAM + RESTORE.md | 5 | `zhy-infra` | Veto 4 |
| INFRA-3 — `.env.prod.example` Bagisto | 1 | `zhy-core-api` | B11/B12 resolvidos |
| **Total Dev 1** | **26 SP** | | |

**Dev 2 — Frontend/BFF:**

| Story / Task | SP | Repo | Notas |
|---|---|---|---|
| OPS-1.4 — `Dockerfile.prod` storefront | 2 | `zhy-storefront-crm` | Multi-stage, `npm run build` |
| OPS-2.2 — Sentry + Pino no Next.js | 2 | `zhy-storefront-crm` | `sentry.client.config.ts` + `src/lib/logger.ts` |
| OPS-2.3 — `GET /api/health` | 0.5 | `zhy-storefront-crm` | Status 200 = OK |
| OPS-3.1 — CI storefront (lint, build, test) | 3 | `zhy-storefront-crm` | V12 expandido |
| OPS-3.4 — CD storefront → GHCR | 2 | `zhy-storefront-crm` | Veto 1 |
| B13 — Verificar `remotePatterns` em prod (validar) | 0.5 | `zhy-storefront-crm` | Teste real com imagens CDN |
| B14 — `.env.example` — verificar completude | 0.5 | `zhy-storefront-crm` | SEC-3.5 |
| **Total Dev 2** | **10.5 SP** | | |

**Critério de saída Sprint 4:** Push `main` → imagem no GHCR ✅ | VPS deploy com `deploy.sh` sem build ✅ | Backup MySQL rodando às 02:00 UTC ✅ | HTTPS em ambos os domínios ✅

---

### Sprint 5 — Semana 6: Catálogo + Variantes + Performance {#sprint-5}

> **Objetivo:** Catálogo completo (menu, PLP com filtros, busca, breadcrumbs) + variantes de produto + ISR tuning.

**Dev 1 — Backend/Infra:**

| Story / Task | SP | Repo | Notas |
|---|---|---|---|
| OPS-2.1 — Sentry no Bagisto | 2 | `zhy-core-api` | Config + `.env` |
| OPS-2.4 — `GET /api/v1/status` health | 0.5 | `zhy-core-api` | Para healthcheck Docker |
| B11 — Variáveis de ambiente de prod no Bagisto | 1 | `zhy-core-api` | `.env.prod` completo |
| **Total Dev 1** | **3.5 SP** | | |

**Dev 2 — Frontend/BFF:**

| Story / Task | SP | Repo | Notas |
|---|---|---|---|
| CAT-1 — Menu de categorias dinâmico | 3 | `zhy-storefront-crm` | Server Component, ISR 1h |
| CAT-2 — PLP com filtros (preço + atributos) | 6 | `zhy-storefront-crm` | `min_price`, `max_price`, query params |
| CAT-3 — Busca de produtos | 4 | `zhy-storefront-crm` | `src/app/busca/page.tsx` |
| CAT-4 — Breadcrumbs dinâmicos | 2 | `zhy-storefront-crm` | `Breadcrumbs.tsx` |
| PROD-1 — PDP produto simples + galeria | 4 | `zhy-storefront-crm` | ISR 5 min |
| PROD-2 — PDP variantes configuráveis | 6 | `zhy-storefront-crm` | `VariantSelector.tsx` + price update |
| INFRA-4 — ISR revalidation tuning | 1 | `zhy-storefront-crm` | PLP 5min, PDP 5min, categorias 60min |
| INFRA-5 — On-demand revalidation endpoint | 2 | `zhy-storefront-crm` | `POST /api/revalidate` |
| **Total Dev 2** | **28 SP** | | |

**Critério de saída Sprint 5:** Menu de categorias com dados reais ✅ | Filtros PLP funcionando ✅ | Variantes selecionáveis no PDP ✅ | ISR validada em staging ✅

---

### Sprint 6 — Semana 7: Checkout Completo + LGPD {#sprint-6}

> **Objetivo:** Checkout multi-step completo com todos os métodos MP + página de sucesso + início da LGPD.

**Dev 1 — Backend/Infra:**

| Story / Task | SP | Repo | Notas |
|---|---|---|---|
| B17+B18 validação final MP | 2 | `zhy-core-api` | Pagamento real em staging |
| B19+B20 carrier + endereço frete | 2 | `zhy-core-api` | Verificar cotação real |
| B21 SMTP em produção | 1 | `zhy-core-api` | Email de confirmação de pedido |
| SEC-4.1 — Endpoint de exclusão de dados LGPD | 4 | `zhy-core-api` | Backend endpoint |
| **Total Dev 1** | **9 SP** | | |

**Dev 2 — Frontend/BFF:**

| Story / Task | SP | Repo | Notas |
|---|---|---|---|
| CART-1 — Carrinho completo (UI + persistence) | 4 | `zhy-storefront-crm` | `CartContext.tsx` + `CartPage` |
| CART-3 — Checkout multi-step completo | 10 | `zhy-storefront-crm` | Address + Shipping + Payment + Review |
| CART-4 — Página de sucesso do pedido | 3 | `zhy-storefront-crm` | Número, itens, total, próximos passos |
| SEC-4.1 — UI de exclusão de dados LGPD | 2 | `zhy-storefront-crm` | `/conta/privacidade` |
| SEC-4.3 — Cookie consent banner | 3 | `zhy-storefront-crm` | `CookieConsent.tsx` |
| **Total Dev 2** | **22 SP** | | |

**Critério de saída Sprint 6:** Checkout end-to-end funcional (PIX + boleto + cartão) ✅ | Página de sucesso ✅ | LGPD básica ✅

---

### Sprint 7 — Semana 8: Go-Live Validation + Deploy de Produção {#sprint-7}

> **Objetivo:** Verificação completa V1–V14 em ambiente de produção real + deploy definitivo + smoke tests pós-deploy.

**Dev 1 — Backend/Infra:**

| Story / Task | SP | Repo | Notas |
|---|---|---|---|
| Deploy final VPS (B4/B5: VPS + DNS) | 4 | VPS + DNS | Domínio de produção configurado |
| `./deploy.sh` em produção | 2 | `zhy-infra` | Pull GHCR + health check |
| Verificar backup S3 rodando | 1 | AWS S3 | Log do cron + arquivo no bucket |
| V13 — PIX em produção | 3 | Produção | Webhook MP apontando para prod |
| V14 — Tests em produção | 2 | `zhy-core-api` | `php artisan test` na imagem prod |
| **Total Dev 1** | **12 SP** | | |

**Dev 2 — Frontend/BFF:**

| Story / Task | SP | Repo | Notas |
|---|---|---|---|
| V1–V12 — Verificação completa em produção | 10 | Produção | Todas as features testadas com dados reais |
| B22 — UI/UX final das páginas de auth | 3 | `zhy-storefront-crm` | Review visual Identity ZHY |
| **Total Dev 2** | **13 SP** | | |

**Critério de saída Sprint 7 = 🚀 GO LIVE:**
- V1–V14 todos verificados ✅ em produção
- PIX SSE funcional com webhook MP apontando para `https://app.seudominio.com.br/api/webhooks/mercadopago`
- Backup MySQL S3 funcionando
- CI/CD pipeline verde
- Sem erros críticos em Sentry nas primeiras 24h

---

### Sprint 8+ — Backlog Pós-Launch {#sprint-8}

> Priorização junto ao Product Owner após go-live. Sem sprint fixo — entrar no backlog e priorizar por impacto.

| Story | EPIC | Resp. | SP Estimado |
|---|---|---|---|
| PROD-3 — Avaliações de produto | PROD | Dev 2 | 5 SP |
| PROD-4 — Produtos relacionados | PROD | Dev 2 | 3 SP |
| AUTH-7 — Wishlist | AUTH | Dev 2 | 4 SP |
| SHIP-4 — Frete grátis por valor mínimo | SHIP | Dev 1 | 2 SP |
| CART-2 — Cupom de desconto | CART | Dev 2 | 3 SP |
| INFRA-6 — E2E tests (Playwright) | INFRA | Dev 2 | 8 SP |
| SEC-4.2 — Export de dados LGPD | SEC | Dev 1 | 4 SP |
| CAT-5 — SEO: meta tags por categoria/produto | CAT | Dev 2 | 3 SP |

---

## Jornadas do Cliente Mapeadas {#jornadas}

> Fluxo end-to-end por tipo de usuário. Todos os pontos de integração API estão documentados.

### Jornada 1 — Descoberta e Navegação

```
Homepage → CategoryNav (ISR 1h) → PLP (ISR 5min + filtros)
  → PDP (ISR 5min + variantes) → AddToCartButton
```
**Endpoints:** `GET /api/v1/categories` → `GET /api/v1/products?category_id={id}` → `GET /api/v1/products/{id}` → `POST /api/checkout/cart`

### Jornada 2 — Compra como Convidado (Guest Checkout)

```
PDP → Carrinho → Checkout Step 1 (endereço + CEP ViaCEP)
  → Step 2 (frete por CEP) → Step 3 (pagamento: boleto/PIX/cartão)
  → Step 4 (review) → POST /api/checkout/onepage/orders → Sucesso
```
**Endpoints:** `POST /api/checkout/cart` → `POST /api/checkout/onepage/addresses` → `POST /api/checkout/cart/estimate-shipping-methods` → `POST /api/checkout/onepage/shipping-methods` → `POST /api/checkout/onepage/payment-methods` → `POST /api/checkout/onepage/orders`

### Jornada 3 — Compra Autenticada

```
Login (/api/auth/login → iron-session) → Conta → Perfil/Endereços
  → Carrinho → Checkout (dados pré-preenchidos) → Pedido
  → /conta/pedidos (histórico)
```
**Endpoints:** `POST /api/v1/customer/login` (server-side BFF) → `GET /api/v1/customer/get` → `GET /api/v1/customer/addresses` → `GET /api/v1/customer/orders`

### Jornada 4 — Pagamento via PIX (Mercado Pago) [Veto 2 + Veto 3]

```
Checkout → Payment Step (método PIX) → POST /orders → QR Code + PIX key
  → /pedido/pix/{orderId} (EventSource conecta SSE /api/pix/sse/{orderId})
  → Webhook MP → Bagisto → Redis PUBLISH "pix:confirmed:{orderId}"
  → SSE entrega "pix_confirmed" ao browser → redirect /pedido/sucesso/{id}
```
**Stack:** Mercado Pago SDK → `ProcessMercadoPagoPayment` Job → `Redis::publish()` → `GET /api/pix/sse/{orderId}` (ioredis subscriber) → `EventSource` no browser

> ❌ **Anti-padrão eliminado (Veto 3):** `usePixPolling()` com `setInterval(10000)` — nunca usar.

### Jornada 5 — Recuperação de Senha

```
/esqueci-senha → POST /api/shop/v1/customer/forgot-password
  → Email (B21: SMTP real) → Link de reset → /redefinir-senha?token=...
  → POST /api/v1/customer/reset-password → Login automático
```

### Jornada 6 — Deploy Contínuo (Dev) [Veto 1]

```
git push main → GitHub Actions CI (lint + build + test)
  → se CI verde → CD dispara → docker build → push ghcr.io/mzgui/...
  → deploy.sh (VPS) → docker pull (sem build) → docker compose up --no-build
  → health check → 🟢 online
```
> ❌ **Anti-padrão eliminado (Veto 1):** `ssh vps "docker compose up -d --build"` — nunca usar.

---

## Pendências e Decisões em Aberto {#pendencias}

> Issues técnicas e de negócio que precisam de decisão antes ou durante a sprint correspondente.

### Decisões de Negócio

| # | Decisão | Impacto | Sprint |
|---|---|---|---|
| B22 | Design das páginas de auth (login, register, esqueci-senha) — aprovação de UI/UX | AUTH-1/2/3 — Sprint 1 | Sprint 0 |
| B23 | Domínio de produção definitivo | OPS-1 (nginx prod.conf, Certbot) | Sprint 4 |
| B24 | Valor mínimo para frete grátis (R$ ?) | SHIP-4 — Sprint 8+ | Sprint 7 |
| B25 | Parcelamento sem juros até quantas vezes? | PAY-BR-4 — Sprint 2 | Sprint 2 |

### Bloqueadores Técnicos Distribuídos por Sprint

| # | Bloqueador | Sprint | Dev | Status |
|---|---|---|---|---|
| B1 | `docker-compose.prod.yml` não existe | Sprint 4 | Dev 1 | OPS-1.1 |
| B2 | `nginx/conf.d/prod.conf` não existe | Sprint 4 | Dev 1 | OPS-1.5 |
| B3 | Let's Encrypt não configurado no VPS | Sprint 4 | Dev 1 | OPS-1.6 |
| B4 | VPS de produção não provisionado | Sprint 7 | Dev 1 | Pre-go-live |
| B5 | DNS não apontado para VPS de produção | Sprint 7 | Dev 1 | Depende de B23 |
| B6 | `Dockerfile.dev` usa `php artisan serve` (não para prod) | Sprint 0 | Dev 1 | SEC-1 + OPS-1.3 |
| B7 | `QUEUE_CONNECTION=sync` — jobs PIX não são enfileirados | Sprint 0 | Dev 1 | SEC-1 |
| B8 | `CancelExpiredPixOrders` nunca agendado no Kernel | Sprint 0 | Dev 1 | SEC-1 |
| B9 | Supervisor não configurado — sem queue worker em prod | Sprint 2 | Dev 1 | OPS-1.2 |
| B10 | `APP_KEY` de staging/produção não gerada | Sprint 0 | Dev 1 | `php artisan key:generate` |
| B11 | `.env.prod` do Bagisto não existe | Sprint 4 | Dev 1 | INFRA-3 |
| B12 | `APP_ENV=local` em staging | Sprint 0 | Dev 1 | Mudar para `production` |
| B13 | `remotePatterns` ausente — `<Image>` quebra com CDN | Sprint 0 | Dev 2 | SEC-1.4 |
| B14 | `.env.example` do storefront não existe | Sprint 2 | Dev 2 | SEC-3.5 |
| B15 | `.env` commitado historicamente — rotacionar todas as credenciais | Sprint 0 | Dev 1 | Urgente |
| B16 | `deploy.sh` não existe — deploy manual via SSH | Sprint 4 | Dev 1 | OPS-3.5 |
| B17 | `mercadopago/dx-php` não instalado | Sprint 1 | Dev 1 | PAY-BR-1 |
| B18 | Nenhum controller de criação de pagamento MP | Sprint 1 | Dev 1 | PAY-BR-1 |
| B19 | Carrier de frete não configurado no Bagisto | Sprint 3 | Dev 1 | SHIP-1/5 |
| B20 | Endereço de origem do vendedor não cadastrado | Sprint 3 | Dev 1 | SHIP-5 |
| B21 | SMTP de produção não configurado (Mailpit só no dev) | Sprint 3 | Dev 1 | Mailgun/SES/Locaweb |

### Decisões Já Tomadas (Referência)

| # | Decisão | Veto |
|---|---|---|
| ✅ Gateway de pagamento | **Mercado Pago exclusivo** — Stripe desabilitado | Veto 2 |
| ✅ Deploy | **GHCR + docker pull** — sem build na VPS | Veto 1 |
| ✅ Notificação PIX | **SSE + Redis Pub/Sub** — sem polling | Veto 3 |
| ✅ Backup de dados | **mysqldump → S3 STANDARD_IA + KMS** | Veto 4 |
| ✅ Sessão no frontend | **iron-session** (`zhy_session` cookie httpOnly, encrypted) | SEC-2 |
| ✅ Auth backend | **Sessão Laravel** via BFF — Sanctum tokens descartados após login | SEC-2 |

---

*Documento gerado em Maio/2026. Scrum Edition v3.0 — Mantido por Dev 1 e Dev 2.*
