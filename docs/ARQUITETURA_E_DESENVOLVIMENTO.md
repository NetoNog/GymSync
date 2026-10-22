# GymSync — Arquitetura, Tecnologias e Guia de Desenvolvimento

> **GymSync** é uma distribuição personalizada (*distro*) baseada no projeto de código aberto [openGym](https://github.com/DuarteSantos8/openGym), desenvolvido originalmente por Duarte Santos e colaboradores sob licença **AGPL-3.0-or-later**.
> 
> Esta versão introduz uma nova identidade visual orientada a alta performance e estética esportiva moderna, combinando **tons escuros de grafite/obsidiana (`#090a0f`, `#12141c`, `#181a24`)** com **laranja vibrante neon (`#ff6a00`, degradê `#ff7a18` → `#ff5200`)**, tipografia atlética aprimorada (**Plus Jakarta Sans** e **Outfit**), além de refinamentos em componentes e experiência do usuário.

---

## 1. Visão Geral do Projeto

O **GymSync** é uma aplicação web progressiva (PWA - Progressive Web App) auto-hospedada (*self-hosted*), voltada para planejamento de treinos, acompanhamento de cargas e evolução corporal.

### Pilares Fundamentais:
1. **Soberania dos Dados**: Seus treinos, histórico de cargas, fotos de progresso e peso corporal pertencem 100% a você. Não há telemetria, anúncios, assinaturas ou dependência de servidores terceiros.
2. **Offline-First & PWA**: O aplicativo funciona offline, pode ser instalado na tela de início de celulares (Android/iOS) ou PCs e sincroniza automaticamente via rede local ou remota quando há conexão.
3. **Autenticação Segura com Passkeys (WebAuthn)**: Login biométrico (Face ID, Touch ID, biometria digital do Android ou chave de segurança física FIDO2) sem necessidade de senhas memorizadas, com opção adicional de senha por usuário.
4. **Design Moderno (Dark Graphite & Vibrant Orange)**: Visual imersivo com contraste calibrado para academias e ambientes escuros, realces laranjas energéticos e micro-interações refinadas.

---

## 2. Tecnologias Utilizadas

### Frontend
* **React 19**: Biblioteca para renderização declarativa e reativa da interface.
* **Vite**: Ferramenta de build e servidor de desenvolvimento ultra-rápido com suporte a hot-reload (HMR) e empacotamento modular.
* **Zustand (v5)**: Gerenciamento de estado global único (`useStore.js`), desacoplado de frameworks complexos, com sincronização em `localStorage` (`gym_state_v1`) e espelhamento para sistema de arquivos móvel.
* **React Router DOM (v7)**: Roteamento declarativo de páginas (`HashRouter` para compatibilidade total com PWA e ambientes estáticos/Capacitor).
* **Vanilla CSS (Design System Tokens)**: Sistema de design baseado 100% em CSS Custom Properties (`--bg`, `--surface`, `--orange`, `--acc`), sem dependências pesadas de frameworks utilitários como Tailwind ou Bootstrap, garantindo leveza extrema e total controle de contraste e animações.
* **Google Fonts**:
  * *Plus Jakarta Sans*: Leitura de alta densidade em telas móveis e desktop.
  * *Outfit*: Títulos de impacto visual moderno e esportivo.
  * *JetBrains Mono*: Alinhamento tabular para pesos, repetições e cronômetros de descanso.
* **Capacitor (Opcional)**: Envoltório nativo para builds Android (APK) e iOS.

### Backend (API)
* **Node.js (v22)**: Executado de forma nativa e enxuta, sem frameworks intermediários (como Express ou Fastify), utilizando diretamente o módulo embutido `node:http`.
* **@simplewebauthn/server**: Biblioteca padrão da indústria para validação criptográfica de credenciais WebAuthn / Passkeys.
* **web-push**: Gerenciamento de notificações push web criptografadas via protocolo VAPID para alertas de fim de descanso e lembretes de treino.
* **Armazenamento Atômico em Arquivos JSON**: Não requer banco de dados relacional pesado (MySQL, PostgreSQL) nem NoSQL complexo. Toda a persistência é feita por escrita atômica em arquivos (`write-temp-then-rename`) no diretório configurado (`DATA_DIR`):
  * `db.json`: Usuários, credenciais de passkeys, assinaturas de push e códigos de convite.
  * `state-<uid>.json`: Estado completo dos treinos de cada usuário.
  * `audit.log`: Log em formato JSONL para eventos de segurança.

### Infraestrutura e Deployment
* **Docker & Docker Compose**: Orquestração multi-container para execução com um único comando.
* **Nginx (Alpine Unprivileged)**: Servidor web de alta performance atuando como:
  * Servidor estático dos arquivos compilados do frontend React.
  * *Reverse Proxy* transparente direcionando requisições `/api/*` para o serviço de backend Node.js.
  * Distribuidor dos ativos multimídia dos exercícios (vídeos, animações WebP e imagens).

---

## 3. Arquitetura do Sistema

```
                  ┌──────────────────────────────────────────────┐
                  │              Cliente (Browser / PWA)         │
                  │   React 19 + Zustand + Vanilla CSS Tokens    │
                  └───────────────────────┬──────────────────────┘
                                          │
                     Requisições HTTP / HTTPS (Porta 8080)
                                          │
                  ┌───────────────────────▼──────────────────────┐
                  │                 Nginx Container               │
                  │        (ghcr.io/duartesantos8/opengym-web)   │
                  ├───────────────────────┬──────────────────────┤
                  │  Arquivos Estáticos   │  Proxy Reverso /api  │
                  │  (HTML, CSS, JS, PWA) │                      │
                  └───────────────────────┼──────────────────────┘
                                          │
                               Proxy interno :3000
                                          │
                  ┌───────────────────────▼──────────────────────┐
                  │              API Node.js Container           │
                  │   HTTP Router Nativo + SimpleWebAuthn        │
                  └───────────────────────┬──────────────────────┘
                                          │
                              Leitura / Escrita Atômica
                                          │
                  ┌───────────────────────▼──────────────────────┐
                  │            Volume de Dados Host              │
                  │     ./data/db.json  |  ./data/state-*.json   │
                  └──────────────────────────────────────────────┘
```

### Componentes Principais

#### 1. Camada de Apresentação (Frontend)
* Localização: `frontend/src/`
* **`store/useStore.js`**: Estado persistido central do aplicativo contendo rotinas, histórico de treinos, peso corporal, perfil e configurações de UI. Sincroniza em segundo plano com o servidor via `pushState` debounced (com controle de concorrência por carimbo temporal e reconciliação em três vias).
* **`views/`**: Telas da aplicação (`Home`, `Workout`, `Plan`, `History`, `Stats`, `Library`, `Settings`, `Login`).
* **`components/`**: Componentes reutilizáveis como `TabBar` (navegação inferior), `RestTimer` (cronômetro flutuante com feedback visual e sonoro), `LineChart` (gráficos SVG sem bibliotecas pesadas), `BodyMap` (mapa muscular anatômico interativo) e `Modals` (sistema de *bottom sheets* com gesto de arrastar para fechar).
* **`lib/`**: Funções puras com testes unitários cobrindo o motor de progressão de carga, cálculo de 1RM, controle de esforço (RIR/RPE), rotações de treino e formatação.

#### 2. Camada de Serviço e Autenticação (API)
* Localização: `api/server.js`
* O servidor responde a rotas pré-mapeadas (`routes['METHOD /path']`) com baixa latência e consumo mínimo de memória RAM (< 50 MB).
* A sessão do usuário é assinada utilizando cookies HTTP-only com HMAC SHA-256 gerado automaticamente na primeira inicialização (`DATA_DIR/secret`).

#### 3. Servidor Web & Proxy (Nginx)
* Localização: `web/`
* `web/nginx.conf.template` injeta variáveis de ambiente (`PORT`, `NGINX_PORT`, `BACKEND`) dinamicamente na inicialização do container.
* Oferece suporte completo a PWA com `Service Worker` e cache otimizado com cabeçalhos de expiração adequados para fontes, mídias e chunks Javascript.

---

## 4. O Redesign GymSync

O redesign foi desenvolvido seguindo critérios modernos de estética e contraste para usabilidade em treinos:

| Elemento | Antes (openGym Padrão) | Agora (GymSync Distro) |
| :--- | :--- | :--- |
| **Fundo da Aplicação** | `#000000` preto plano | `#090a0f` grafite profundo (obsidiana com subtom frio) |
| **Superfície dos Cards** | `#1c1c1e` cinza comum | `#181a24` grafite escuro com borda translúcida e sombra de profundidade |
| **Cor de Destaque (Accent)** | Verde claro `#30d158` (*lime*) | **Laranja vibrante elétrico `#ff6a00`** com degradê de energia |
| **Botão Central de Treino** | Disco verde | Disco circular com degradê laranja (`#ff7a18` → `#ff5200`), glow e pulso dinâmico durante treinos |
| **Tipografia** | Fontes padrão do sistema | **Plus Jakarta Sans** (interface) e **Outfit** (títulos atléticos) |
| **Identidade / Logotipo** | Texto simples com ícone | Emblema de haltere com aura de luz em degradê e tipografia estilizada `GymSync` |
| **Navegação Inferior (TabBar)** | Barra cinza básica | Vidro grafite fosco (`backdrop-filter: blur(16px)`) com ícones laranjas ativos |

---

## 5. Como Executar o Projeto

### Opção 1: Usando Docker Compose (Recomendado)

1. Certifique-se de que o Docker e Docker Compose estão instalados.
2. Copie o arquivo de variáveis de ambiente:
   ```bash
   cp .env.example .env
   ```
3. Inicie os containers compilando as imagens locais:
   ```bash
   docker compose up -d --build
   ```
4. Acesse no navegador:
   ```
   http://localhost:8080
   ```

### Opção 2: Modo de Desenvolvimento Local (Node.js)

1. **Backend (API)**:
   ```bash
   cd api
   npm install
   # No Windows PowerShell:
   $env:DATA_DIR="../data"; node server.js
   # No Linux/macOS:
   DATA_DIR="../data" node server.js
   ```
2. **Frontend (Vite)**:
   ```bash
   cd frontend
   npm install
   npm run dev
   ```
3. Acesse `http://localhost:5173`. O Vite repassará chamadas para `http://127.0.0.1:3000`.

---

## 6. Como Sincronizar com o Repositório Upstream (openGym)

Por se tratar de uma distribuição personalizada do [openGym](https://github.com/DuarteSantos8/openGym), você pode incorporar novos recursos e correções de segurança do projeto original mantendo as suas alterações no GymSync:

1. **Configurar o remoto upstream**:
   ```bash
   git remote add upstream https://github.com/DuarteSantos8/openGym.git
   ```

2. **Buscar atualizações do openGym**:
   ```bash
   git fetch upstream
   ```

3. **Mesclar com o GymSync**:
   ```bash
   git checkout main
   git merge upstream/main
   ```
   *(Caso haja conflitos em `index.css` ou `Login.jsx`, preserve as personalizações do GymSync).*

4. **Enviar para o seu repositório**:
   ```bash
   git push origin main
   ```

---

## 7. Licença e Créditos

* O código-fonte original do openGym é de autoria de **Duarte Santos** e colaboradores, sob licença **AGPL-3.0-or-later**.
* As modificações, identidade visual e documentação do **GymSync** são mantidas por [NetoNog](https://github.com/NetoNog) e distribuídas sob os mesmos termos da licença AGPL v3.
* Imagens e ilustrações de exercícios são licenciadas por [Gym visual](https://gymvisual.com) (ver `NOTICE.md`).
