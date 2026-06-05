# Deep Linking — Estudo e POC

Projeto de estudo sobre Deep Linking em React Native com Expo Router.

---

## O que é Deep Linking?

Deep Linking é a capacidade de abrir um app mobile em uma tela específica a partir de uma URL externa — seja por outro app, notificação, e-mail, SMS ou browser.

---

## Tipos de Deep Link

### 1. Custom URL Scheme

```
deeplinking:///product/42
```

- Funciona em qualquer app
- Simples de configurar
- **Problema:** qualquer app pode registrar o mesmo scheme — sem garantia de exclusividade
- Não abre no browser se o app não estiver instalado

### 2. Universal Links (iOS) / App Links (Android)

```
https://meudomain.com/product/42
```

- Usa HTTPS — mais seguro e exclusivo (precisa de controle do domínio)
- Abre o app se instalado, senão cai no browser (fallback automático)
- Requer um arquivo de verificação hospedado no servidor:
  - iOS: `apple-app-site-association` em `/.well-known/`
  - Android: `assetlinks.json` em `/.well-known/`

---

## Configuração no Expo

### `app.json` — scheme e intent filters

```json
{
  "expo": {
    "scheme": "deeplinking",
    "android": {
      "intentFilters": [
        {
          "action": "VIEW",
          "data": [{ "scheme": "deeplinking" }],
          "category": ["BROWSABLE", "DEFAULT"]
        }
      ]
    }
  }
}
```

O campo `scheme` registra o custom URL scheme. Os `intentFilters` ensinam o Android a redirecionar links para o app.

---

## Roteamento com Expo Router

Com Expo Router, **não existe arquivo central de rotas**. O mapeamento é automático pela estrutura de arquivos:

```
app/
  _layout.tsx           → layout raiz
  (tabs)/
    index.tsx           → deeplinking:///
    explore.tsx         → deeplinking:///explore
  product/
    [id].tsx            → deeplinking:///product/:id
  modal.tsx             → deeplinking:///modal
```

### Parâmetros dinâmicos

```tsx
// app/product/[id].tsx
import { useLocalSearchParams } from 'expo-router';

export default function ProductScreen() {
  const { id } = useLocalSearchParams();
  return <Text>Produto: {id}</Text>;
}
```

---

## Receber links no app

### App estava fechada (cold start)

```tsx
import * as Linking from 'expo-linking';

const url = await Linking.getInitialURL();
```

### App estava em background (warm start)

```tsx
Linking.addEventListener('url', ({ url }) => {
  // tratar o link
});
```

Com Expo Router isso é tratado automaticamente — o router intercepta a URL e navega para a rota correspondente.

---

## Expo Go vs Development Build

| | Expo Go | Development Build |
|---|---|---|
| Custom scheme (`deeplinking://`) | Não funciona | Funciona |
| Universal/App Links | Não funciona | Funciona |
| Navegação interna (`router.push`) | Funciona | Funciona |
| Ideal para | Prototipagem rápida | Testar deep links reais |

Para buildar o development build:

```bash
npm run android   # instala no dispositivo via USB
npm run ios       # instala no simulador/dispositivo iOS
```

---

## Testando Deep Links

### No dispositivo físico (development build)

```bash
# Android — via adb
adb shell am start -W -a android.intent.action.VIEW -d "deeplinking:///product/42"

# iOS — via xcrun
xcrun simctl openurl booted "deeplinking:///product/42"
```

> `adb` fica em `~/Library/Android/sdk/platform-tools/adb`. Adicione ao PATH:
> ```bash
> export ANDROID_HOME=$HOME/Library/Android/sdk
> export PATH=$PATH:$ANDROID_HOME/platform-tools
> ```

### No próprio app

```tsx
import { Linking } from 'react-native';

<Button
  title="Testar deeplink"
  onPress={() => Linking.openURL('deeplinking:///product/42')}
/>
```

---

## Edge Cases importantes

- **App não instalada:** o link não abre — use Universal/App Links para ter fallback no browser
- **Link chega antes da navegação estar pronta:** precisar enfileirar e processar após o mount
- **Autenticação:** se o destino exige login, redirecione para o login e após autenticar volte ao destino original

---

## Estrutura deste projeto

```
app/
  (tabs)/
    index.tsx       — home com botões de teste
    explore.tsx     — aba explorar
  product/
    [id].tsx        — tela de produto recebe ID via URL
  modal.tsx         — modal acessível via deeplink
  _layout.tsx       — layout raiz com Stack navigator
app.json            — scheme e intentFilters configurados
```
