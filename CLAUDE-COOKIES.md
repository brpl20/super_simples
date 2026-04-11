# Cookie Consent + GA4 Consent Mode — Guia de Replicacao

Guia para adicionar cookie consent com GA4 Consent Mode v2 em sites Gatsby.
Implementado pela primeira vez no site integracaogarantidora.com.br em abril/2026.

## Decisoes de design

- **Minimo necessario** para conformidade com Google Consent Mode v2
- Banner simples com dois botoes: "Aceitar" e "Recusar" (sem categorias granulares)
- Barra fixa no rodape da pagina
- Preferencia salva no `localStorage` (chave: `cookie-consent`, valores: `granted` ou `denied`)
- Banner nao aparece mais apos a escolha do usuario
- Visitante que volta ao site com consentimento salvo ja carrega com `granted`
- Sem dependencias externas (nenhuma lib de cookies/CMP)
- Cores do banner devem seguir a identidade visual do site

## Como o Google Consent Mode funciona

1. Antes de qualquer script do GA4, voce define o consentimento como **negado por padrao**
2. O GA4 carrega em modo restrito (pings anonimizados, sem cookies)
3. Quando o usuario aceita, voce atualiza o consentimento para **granted**
4. O GA4 passa a funcionar normalmente com cookies
5. Se o usuario recusa, nada muda (ja estava denied)

O Google detecta isso automaticamente — nao precisa configurar nada no painel do GA4.

## Arquivos necessarios (Gatsby)

### 1. `gatsby-ssr.tsx` (novo)

Injeta o script de consent default no `<head>` ANTES do script do gtag.
Usa `onPreRenderHTML` para garantir que o script seja o primeiro elemento no head.

```tsx
import React from "react";
import type { GatsbySSR } from "gatsby";

export const onRenderBody: GatsbySSR["onRenderBody"] = ({
  setHeadComponents,
}) => {
  setHeadComponents([
    <script
      key="cookie-consent-default"
      dangerouslySetInnerHTML={{
        __html: `
          window.dataLayer = window.dataLayer || [];
          function gtag(){dataLayer.push(arguments);}
          gtag('consent', 'default', {
            analytics_storage: 'denied',
            ad_storage: 'denied',
            wait_for_update: 500
          });
          try {
            if (localStorage.getItem('cookie-consent') === 'granted') {
              gtag('consent', 'update', {
                analytics_storage: 'granted'
              });
            }
          } catch(e) {}
        `,
      }}
    />,
  ]);
};

export const onPreRenderHTML: GatsbySSR["onPreRenderHTML"] = ({
  getHeadComponents,
  replaceHeadComponents,
}) => {
  const headComponents = getHeadComponents();
  const consentIndex = headComponents.findIndex(
    (c: any) => c.key === "cookie-consent-default"
  );
  if (consentIndex > 0) {
    const [consentScript] = headComponents.splice(consentIndex, 1);
    headComponents.unshift(consentScript);
    replaceHeadComponents(headComponents);
  }
};
```

**Por que `onPreRenderHTML`?** O `gatsby-plugin-google-gtag` injeta seus scripts no head antes dos nossos `setHeadComponents`. Sem a reordenacao, o `gtag('config', ...)` roda antes do `gtag('consent', 'default', ...)`, o que invalida o consent mode.

### 2. `src/components/CookieBanner.tsx` (novo)

Adapte as cores (`backgroundColor`, `borderTopColor`, `color`) para cada site.

```tsx
import * as React from "react";

declare global {
  interface Window {
    dataLayer: Array<unknown>;
    gtag: (...args: unknown[]) => void;
  }
}

const CookieBanner: React.FC = () => {
  const [visible, setVisible] = React.useState(false);

  React.useEffect(() => {
    try {
      const consent = localStorage.getItem("cookie-consent");
      if (!consent) {
        setVisible(true);
      }
    } catch {
      setVisible(true);
    }
  }, []);

  const handleAccept = () => {
    try {
      localStorage.setItem("cookie-consent", "granted");
    } catch {}
    if (typeof window.gtag === "function") {
      window.gtag("consent", "update", {
        analytics_storage: "granted",
      });
    }
    setVisible(false);
  };

  const handleRefuse = () => {
    try {
      localStorage.setItem("cookie-consent", "denied");
    } catch {}
    setVisible(false);
  };

  if (!visible) return null;

  return (
    <div
      className="fixed bottom-0 left-0 right-0 z-50 flex flex-col sm:flex-row items-center justify-between gap-3 px-4 sm:px-6 py-4 border-t-2"
      style={{
        backgroundColor: "#00253E",  // <- cor de fundo do banner
        borderTopColor: "#CA9C4D",   // <- cor da borda superior
      }}
    >
      <p className="text-white text-sm text-center sm:text-left m-0">
        Utilizamos cookies para analisar o trafego do site. Ao aceitar, voce
        concorda com o uso de cookies analiticos.
      </p>
      <div className="flex gap-3 shrink-0">
        <button
          onClick={handleRefuse}
          className="px-5 py-2 rounded-md text-sm font-semibold bg-transparent cursor-pointer"
          style={{
            border: "1.5px solid #CA9C4D",  // <- cor do botao recusar
            color: "#CA9C4D",
          }}
        >
          Recusar
        </button>
        <button
          onClick={handleAccept}
          className="px-5 py-2 rounded-md text-sm font-bold border-none cursor-pointer"
          style={{
            backgroundColor: "#CA9C4D",  // <- cor do botao aceitar
            color: "#00253E",
          }}
        >
          Aceitar
        </button>
      </div>
    </div>
  );
};

export default CookieBanner;
```

### 3. `gatsby-browser.js` (editar)

Adicione o `wrapPageElement` para renderizar o banner em todas as paginas:

```js
import "./src/styles/global.css";
import React from "react";
import CookieBanner from "./src/components/CookieBanner";

export const wrapPageElement = ({ element }) => (
  <>
    {element}
    <CookieBanner />
  </>
);
```

Se o arquivo ja tiver um `wrapPageElement`, adicione o `<CookieBanner />` dentro do JSX existente.

### 4. `gatsby-config.ts` (editar)

No plugin `gatsby-plugin-google-gtag`, remova `cookie_expires: 0` se existir.
O consent mode controla o ciclo de vida dos cookies agora.

```ts
{
  resolve: "gatsby-plugin-google-gtag",
  options: {
    trackingIds: ["G-XXXXXXXXXX"],  // <- seu ID
    gtagConfig: {
      anonymize_ip: true,
    },
    pluginConfig: {
      head: true,  // IMPORTANTE: precisa ser true
    },
  },
},
```

## Checklist para replicar em outro site

1. [ ] Copiar `gatsby-ssr.tsx` para a raiz do projeto (nao precisa mudar nada)
2. [ ] Copiar `CookieBanner.tsx` para `src/components/` e adaptar as cores
3. [ ] Editar `gatsby-browser.js` adicionando o `wrapPageElement`
4. [ ] Verificar `gatsby-config.ts`: remover `cookie_expires: 0`, confirmar `head: true`
5. [ ] Build e verificar no HTML que consent default vem ANTES de `gtag('config', ...)`
6. [ ] Deploy
7. [ ] Verificar no GA4 Real-time que os hits estao chegando apos aceitar

## Como verificar no navegador

- **DevTools > Application > Local Storage** — deve ter `cookie-consent` com valor `granted` ou `denied`
- **DevTools > Console** — rodar `dataLayer` e procurar entries de consent
- **DevTools > Network** — filtrar por `collect`, parametro `gcs=G111` = granted, `gcs=G100` = denied
- **View Source (Ctrl+U)** — confirmar que o script de consent aparece antes do `googletagmanager.com/gtag/js`

## Fora de escopo (decisoes conscientes)

- Categorias granulares de cookies (so temos analytics, nao precisa)
- Pagina de politica de privacidade (pode ser adicionada depois)
- Plataformas de consentimento de terceiros (CookieBot, iubenda, etc.)
- Cookies de marketing/ads (nao usados)
- Botao para revogar consentimento depois (pode ser adicionado no footer se necessario)
