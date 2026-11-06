# Protocolo de Monetización PAI

Infraestructura para que una PYME sin activos fijos hipotecables convierta su **cartera de derechos de cobro** en garantía verificable, y obtenga financiamiento contra ella.

Hackathon **Track Arbitrum** · categoría **DeFi / RWA**

> **La tesis:** la cadena no reemplaza al abogado ni al registro público — reemplaza la necesidad de confiar en el operador de la plataforma. El monto prestable deja de ser un número que el backend afirma, y pasa a ser una función determinista que el prestamista puede recomputar.

**El test decisivo:** si un jurado quita la blockchain y la solución sigue funcionando igual, el proyecto no califica en DeFi/RWA — califica en gestión documental. Por eso lo que vive on-chain es la **ruta crítica de confianza**: valorización, custodia del colateral y dinero. No el expediente.

---

## 🧭 Empieza por aquí

| Si eres…                | Ve a                                                                                                                                                     |
| ----------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Jurado o evaluador**  | [Estado real](#-estado-real-qué-funciona-hoy) → [Los dos momentos](#-los-dos-momentos-que-importan) → [Verificar en 5 minutos](#-verificar-en-5-minutos) |
| **Desarrollador nuevo** | [Arranque](#-arranque) → [Arquitectura](#-arquitectura-la-frontera-web2--web3) → [Agregar una feature](#-agregar-una-feature-receta)                     |
| **Quien despliega**     | [`.env` completo](#-el-env-completo) → [Deploy en Coolify](#-deploy-en-coolify)                                                                          |

Documentos largos: [`docs/vision-general.md`](docs/vision-general.md) (mapa completo del sistema) · [`docs/referencia-pai-arbitrum.md`](docs/referencia-pai-arbitrum.md) (caso de uso, rúbrica, guion de demo) · [`docs/arquitectura.md`](docs/arquitectura.md) (frontera Web2 ↔ Web3) · [`docs/design/`](docs/design/) (sistema visual del panel).

---

## 📊 Estado real: qué funciona hoy

Esta tabla es deliberadamente honesta. Un jurado que descubre solo que una demo estaba maquillada deja de creer todo lo demás, incluido lo que sí era cierto.

| Pieza                           | Estado                  | Detalle                                                                                                          |
| ------------------------------- | ----------------------- | ---------------------------------------------------------------------------------------------------------------- |
| **Hoja canónica de Merkle**     | ✅ Funciona             | `packages/merkle` con vectores dorados. Solidity y TypeScript hashean igual, verificado por tests en ambos lados |
| **Divulgación selectiva**       | ✅ Funciona             | Multiproof real end-to-end. Selección, root, proof, flags, verificación                                          |
| **Motor de borrowing base**     | ✅ Funciona             | `packages/borrowing-base`, aritmética entera en unidades menores, con vectores dorados                           |
| **Panel de operación**          | ✅ Funciona             | 10 rutas + landing pública, sistema visual completo, responsive en móvil                                         |
| **Contratos Solidity**          | ✅ Escritos y testeados | 7 contratos, 67 tests en Foundry, con script de deploy que verifica el cableado                                  |
| **`packages/evm`**              | ✅ Funciona             | ABIs y tipos generados desde `chain/`, 88 tests                                                                  |
| **Adaptador de Arbitrum**       | ✅ Implementado         | `ArbitrumChainAdapter` lee la cadena con viem. Ya no es un stub                                                  |
| **Firma desde wallet**          | ✅ Funciona             | La API prepara el _intent_ y el navegador lo firma por EIP-1193. **El backend no guarda claves**                 |
| **Despliegue en Sepolia**       | 🟡 Pendiente            | Los contratos existen y el script también; faltan las direcciones en el `.env`                                   |
| **Motor en Stylus (Rust)**      | 🔴 No existe            | `chain/stylus/` está vacío. `BorrowingBaseEngine` hoy es **Solidity**, no Rust                                   |
| **Cuenta inteligente ERC-4337** | 🔴 No existe            | Login con Google sí funciona; la smart account en segundo plano, no                                              |

**Consecuencia visible:** las pantallas están cableadas a endpoints reales, pero mientras no existan las direcciones desplegadas muestran un estado vacío que dice **qué dato falta y qué lo desbloquea**, en lugar de una cifra de relleno. Esa fue una decisión, no una limitación:

> _Vacío con explicación es mejor que falso._

### Tres cosas de la maqueta que rechazamos a propósito

El diseño de referencia venía con datos simulados. Portar esos tres habría sido más rápido, y peor:

| Lo que hacía la maqueta                    | Por qué no se portó                                                                               |
| ------------------------------------------ | ------------------------------------------------------------------------------------------------- |
| Contador de bloques `+1` cada 2.4s         | No hay RPC conectado. Un ticker falso corriendo en vivo delante de un jurado no se puede defender |
| Tamaño del proof = `ceil(ocultas / 4) + 8` | Es aritmética inventada, sin relación con un multiproof real. Sale de `proof.length`              |
| «Este número lo calculó Stylus»            | El motor no existe. El panel muestra la misma cifra y dice de dónde salió de verdad               |

---

## 🎯 Los dos momentos que importan

De `docs/referencia-pai-arbitrum.md`:

> **Regla de oro:** mostrar el **selector de divulgación** y el **desglose del borrowing base** aunque haya que cortar otra cosa. Son los dos momentos donde un jurado ve algo que no vio en las otras 40 demos.

### 1. Divulgación selectiva — `/divulgacion`

El prestamista necesita valorar la cartera. La PYME no quiere entregarle su lista de clientes.

Se eligen las cuotas a mostrar y se construye un **multiproof de Merkle real**. El prestamista recibe solo las hojas divulgadas y puede probar que pertenecen al expediente certificado — sin ver las demás ni sus contrapartes.

**El argumento entero está en un detalle:** el `merkleRoot` **no cambia** con la selección. El panel lo hace explícito y cuenta cuántas selecciones distintas se probaron sin que se moviera. Sin ZK, solo un árbol de Merkle.

### 2. Recómputo del borrowing base — `/borrowing-base`

> _«Este número no le pedimos que lo crea. Que lo recompute.»_

El desglose línea a línea: nominal divulgado, descuento por plazo, haircut de morosidad, haircut de concentración, ajuste de continuidad, valor ajustado, base prestable.

**Con la honestidad por delante:** hoy el cálculo corre en el navegador con `@app/borrowing-base` —la misma especificación normativa que el motor Stylus deberá reproducir byte a byte— y la pantalla lo etiqueta como **cálculo local de referencia**. La columna on-chain está construida y vacía. En palabras del propio panel:

> «La insignia MATCH solo significa algo cuando hay dos números de dos fuentes distintas: hoy solo hay uno, y decir MATCH sería comparar la pantalla consigo misma.»

Cuando el motor se despliegue, se enchufa la llamada `view` y la insignia aparece. La pantalla no se rehace.

---

## 🔍 Verificar en 5 minutos

El track exige **evidencia verificable en el repositorio**. Esto es lo que puedes reproducir sin desplegar nada:

```bash
pnpm install
pnpm --filter "./packages/*" build
pnpm test          # 327 tests: merkle, borrowing-base, evm, api y dominio del panel
```

| Qué quieres comprobar                             | Cómo                                                                                                                                         |
| ------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| Solidity y TypeScript hashean **idéntico**        | `cd chain && forge test --match-contract GoldenVectors -vv` — lee los mismos `packages/merkle/fixtures/golden-vectors.json` que usa el front |
| El multiproof **verifica de verdad**              | `pnpm --filter @app/merkle test` (46 tests)                                                                                                  |
| La aritmética del riesgo es entera y determinista | `pnpm --filter @app/borrowing-base test` (25 tests)                                                                                          |
| Los ABIs coinciden con los contratos              | `pnpm --filter @app/evm test` (88 tests)                                                                                                     |
| El panel se ve y funciona                         | `pnpm --filter @app/web test:e2e` (75 tests, escritorio y móvil)                                                                             |
| Los contratos hacen lo que dicen                  | `cd chain && forge test -vvv` (67 tests)                                                                                                     |

### Qué garantiza el CI en cada PR

| Job                             | Qué prueba                                                                 |
| ------------------------------- | -------------------------------------------------------------------------- |
| `ci`                            | lint · typecheck · build · 327 tests en todos los workspaces               |
| `contracts`                     | `forge fmt --check` · `forge build --sizes` · 67 tests de Solidity         |
| `e2e`                           | 75 tests de Playwright, escritorio **y** móvil, con `axe` de accesibilidad |
| `docker (api)` / `docker (web)` | Construye las **mismas imágenes** que el deploy de producción              |

Los dos últimos existen por una razón concreta: el job `ci` compila sobre el checkout completo del monorepo, así que nunca ejerce el subconjunto de archivos que sí ve Docker. Un CI verde llegó a convivir con un deploy roto. Ahora no puede.

---

## 🏗 Arquitectura: la frontera Web2 ↔ Web3

El reparto es deliberado — **~55% Web2, ~45% Arbitrum**:

| Qué                                 | Dónde                  | Por qué ahí                                                                                            |
| ----------------------------------- | ---------------------- | ------------------------------------------------------------------------------------------------------ |
| Expediente, evidencias, CRUD        | Web2 · NestJS          | Poner un PDF on-chain no aporta nada y cuesta                                                          |
| Registro y ciclo de vida del activo | Solidity               | El estado del activo no puede depender de un flag en Postgres                                          |
| Valorización                        | Stylus (Rust)          | El prestamista tiene que poder recomputar la cifra él mismo                                            |
| Custodia y dinero                   | CollateralVault + USDC | **El dinero nunca pasa por nuestro servidor** — jurídicamente nos saca de ser intermediario financiero |

El repo está diseñado para que las dos mitades avancen **en paralelo, sin bloquearse**. La frontera son **dos artefactos, no un deploy**:

1. **`packages/merkle`** — la codificación canónica de la hoja, en TypeScript, con vectores dorados. Los tests de Solidity reproducen esos mismos vectores byte a byte.
2. **`ChainPort`** — la interfaz que el dominio de la API usa para hablar con la cadena. El adaptador en memoria funciona hoy; el de Arbitrum se enchufa sin tocar el dominio.

> Si los dos lados hashean distinto, el multiproof falla en la demo y nadie se entera hasta el minuto 2:00 del pitch. Por eso los vectores dorados existen antes que los contratos.

**Nunca subes el archivo, ni el contrato, ni datos personales. Solo subes una huella.** El `debtorHash` lleva salt: sin él, el nombre de un deudor se recupera por fuerza bruta.

### Reglas duras

- **SHA-256 para archivos, keccak256 para hojas del árbol.** No son intercambiables.
- **Montos en unidades menores** (enteros). Nada de flotantes en dinero, en ningún lado.
- La API **no firma transacciones de valor**. Firma atestaciones EIP-712 y lee eventos.
- Cuando Postgres y la cadena discrepan, **gana la cadena**.
- `synchronize: false` siempre. El esquema cambia solo con migraciones.

---

## 🚫 Lo que NO se construye

Marketplace · tokens transferibles · oráculos descentralizados · fraccionamiento · DAO.

Está fuera de alcance a propósito. El proyecto se juega en mover la ruta crítica de confianza, no en acumular superficie.

Y una precisión que sostiene la credibilidad del pitch: el contrato **no ejecuta la garantía**. Ante un default, **produce la prueba** que activa la ejecución legal — y esa prueba es lo que hoy toma meses litigar. Responder «el smart contract ejecuta la garantía» sería falso.

---

## 🚀 Arranque

```bash
cp .env.example .env                 # ajusta JWT_SECRET y ADMIN_*
pnpm install
pnpm --filter "./packages/*" build   # la API importa @app/contracts y @app/merkle
pnpm db:up                           # postgres en docker
pnpm migration:run
pnpm seed                            # admin inicial (idempotente)
pnpm dev                             # api en :3000, web en :5173
```

Abre **http://localhost:5173**, inicia sesión con el `ADMIN_EMAIL` / `ADMIN_PASSWORD` de tu `.env` y ve a **Divulgación selectiva**.

Contratos (opcional, necesita [Foundry](https://getfoundry.sh)):

```bash
git submodule update --init --recursive
cd chain && forge test -vv
```

---

## 🔑 El `.env` completo

La fuente autoritativa es `apps/api/src/config/env.validation.ts`: es un esquema Zod que se valida **una sola vez al arrancar**. Si falta algo requerido, la API no levanta.

> **El backend no guarda ninguna clave privada.** No hay `ATTESTOR_PRIVATE_KEY` ni equivalente: las transacciones de valor las firma la wallet del usuario en el navegador. Si en algún momento estás por poner una clave privada aquí para mover dinero, para — eso rompe el argumento jurídico del proyecto.

### Obligatorias siempre

Sin estas la API no arranca, con cualquier `CHAIN_ADAPTER`:

```env
DB_HOST=localhost
DB_USER=app
DB_PASSWORD=app
DB_NAME=pai_arbitrum

# Mínimo 16 caracteres. Genera uno: openssl rand -base64 32
JWT_SECRET=cambiame_por_un_secreto_largo

# Storage S3-compatible (MinIO en local, S3/R2 en producción).
# Desde el módulo de evidencias son obligatorias, no sugerencias.
STORAGE_REGION=us-east-1
STORAGE_BUCKET=pai-evidence
STORAGE_ACCESS_KEY=minioadmin
STORAGE_SECRET_KEY=minioadmin
```

### Obligatorias con `CHAIN_ADAPTER=arbitrum`

Aquí van las direcciones que produce `chain/script/Deploy.s.sol`. **Si falta una sola, la API se niega a arrancar** — a propósito: mejor no levantar que fallar en la primera transacción del demo.

```env
CHAIN_ADAPTER=arbitrum
CHAIN_RPC_URL=https://sepolia-rollup.arbitrum.io/rpc

# Bloque del deploy. Sin él, el indexador escanearía la cadena desde el 0.
CHAIN_DEPLOYMENT_BLOCK=

ASSET_REGISTRY_ADDRESS=
CERTIFICATION_ATTESTOR_ADDRESS=
PAI_CERTIFICATE_ADDRESS=
BORROWING_BASE_ENGINE_ADDRESS=
COLLATERAL_VAULT_ADDRESS=
MOCK_USDC_ADDRESS=
```

Con `CHAIN_ADAPTER=in-memory` (el default) todas son opcionales y la API funciona entera sin cadena: dev, tests y la mitad Web2 de la demo.

### Roles de la demo

Deciden quién ve la cola de atestaciones y quién ve el fondeo. **Sin ellas todos entran como PYME** y la demo de tres certificadores no funciona:

```env
CERTIFIER_EMAILS=contador@ejemplo.com,abogado@ejemplo.com,auditor@ejemplo.com
FUND_EMAILS=fondo@ejemplo.com
```

### Con default, pero fíjalas en producción

```env
NODE_ENV=production
# Solo si Postgres no escucha en el 5432.
DB_PORT=5432
CHAIN_ID=421614
CHAIN_EXPLORER_URL=https://sepolia.arbiscan.io

# Orígenes exactos separados por coma. Nunca `*`: la cookie viaja con credentials.
CORS_ORIGIN=https://pai.cloud.groowtech.com
FRONTEND_URL=https://pai.cloud.groowtech.com

# Cookie de sesión httpOnly. Detrás de HTTPS, `secure` es obligatorio.
COOKIE_SECURE=true
COOKIE_SAMESITE=lax
COOKIE_DOMAIN=.groowtech.com

AUTH_LOCAL_ENABLED=true
JWT_EXPIRES_IN=7d
BCRYPT_ROUNDS=12

# Google OAuth: opcional. Con CLIENT_ID y CLIENT_SECRET el botón se activa solo.
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=
GOOGLE_CALLBACK_URL=https://api.tudominio.com/api/auth/google/callback

# Admin inicial (seed idempotente).
ADMIN_EMAIL=admin@example.com
ADMIN_PASSWORD=Admin1234!
ADMIN_NAME=Admin

# true para MinIO (path-style), false para S3 real.
STORAGE_FORCE_PATH_STYLE=false
# Opcional: solo para MinIO. AWS S3 y R2 lo resuelven por región.
STORAGE_ENDPOINT=
```

> **`API_PORT` no lo definas en Coolify.** El default es 3000 y el healthcheck del Dockerfile está fijado ahí.

### El frontend va aparte: son _build args_

Estas dos **no son variables de runtime**. Se hornean en el bundle al compilar, así que cambiarlas exige reconstruir la imagen:

```env
VITE_API_URL=https://api.tudominio.com
VITE_EXPLORER_TX_URL=https://sepolia.arbiscan.io/tx
```

`VITE_EXPLORER_TX_URL` es la base para enlazar cada hash de transacción al explorador. Si falta, la pantalla de fondeo **compila igual** y el enlace queda vacío: es un `undefined` silencioso, no un fallo de build.

---

## 📦 Estructura

```
├── apps/
│   ├── api/               # NestJS 11 · TypeORM · auth JWT en cookie httpOnly
│   └── web/               # React 19 · Vite 7 · Tailwind v4 · shadcn · TanStack Query
│       └── e2e/           # Playwright: escritorio + móvil, con axe
├── packages/
│   ├── contracts/         # Schemas Zod compartidos api ↔ web (NO son smart contracts)
│   ├── merkle/            # Hoja canónica, árbol y multiproof + vectores dorados
│   ├── borrowing-base/    # Motor de riesgo como especificación normativa
│   ├── evm/               # ABIs y tipos generados desde chain/ (NO editar a mano)
│   └── tsconfig/          # Config TypeScript base
├── chain/
│   ├── src/               # 7 contratos Solidity
│   ├── script/            # Deploy.s.sol: despliega y verifica el cableado
│   └── stylus/            # Rust, todavía vacío
└── docs/
    └── design/            # Sistema visual Nocturne + maqueta de referencia
```

Los ABIs de `packages/evm` se **generan** desde `chain/`, nunca se editan a mano: uno copiado a mano se desincroniza el día del demo.

### ⚠️ El choque de nombres

`packages/contracts` son **schemas Zod**, no smart contracts. Los smart contracts viven en `chain/`. Esto no se cambia: medio monorepo apunta ya a `@app/contracts`.

---

## 🎨 El panel

Sistema visual **Nocturne**, tema oscuro único, portado a los tokens de shadcn/ui. Tres convenciones que hay que respetar al construir pantallas nuevas:

| Regla                                                    | Por qué                                                                                            |
| -------------------------------------------------------- | -------------------------------------------------------------------------------------------------- |
| El acento de Nocturne es la **marca** → va a `--primary` | El `--accent` de shadcn es una superficie de hover. Confundirlos tiñe de lila cada hover del panel |
| Rampas `brand-100..900` e `ink-100..900`                 | Los namespaces `accent` y `neutral` pisarían las paletas nativas de Tailwind en silencio           |
| **`ink-600` nunca para texto**                           | Rinde 4.08:1 sobre el fondo: falla WCAG AA. El texto secundario usa `text-muted-foreground`        |

Los tests e2e **miden los tokens computados**, no comparan píxeles: una regresión de color falla con un número, no con un diff de imagen que alguien aprueba de reojo.

Detalle completo del diseño en [`docs/design/README.md`](docs/design/README.md).

---

## 🛠 Scripts (desde la raíz)

| Comando                                                  | Qué hace                                       |
| -------------------------------------------------------- | ---------------------------------------------- |
| `pnpm dev`                                               | API + web + packages en watch, en paralelo     |
| `pnpm dev:api` / `pnpm dev:web`                          | Solo una app                                   |
| `pnpm build`                                             | Build de todo, en orden topológico             |
| `pnpm lint` / `pnpm typecheck` / `pnpm test`             | Calidad en todos los workspaces                |
| `pnpm --filter @app/web test:e2e`                        | Playwright: escritorio y móvil                 |
| `pnpm db:up` / `pnpm db:down`                            | Postgres en Docker                             |
| `pnpm migration:generate src/database/migrations/Nombre` | Genera migración desde los cambios en entities |
| `pnpm migration:run` / `pnpm migration:revert`           | Aplica / revierte migraciones                  |
| `pnpm seed`                                              | Admin inicial (idempotente)                    |

---

## 🗃 Base de datos: migraciones SIEMPRE

```bash
# 1. Edita o crea un *.entity.ts
# 2. Genera la migración (compara entities vs DB real)
pnpm migration:generate src/database/migrations/AgregaCampoX
# 3. Revisa el SQL generado (¡siempre!)
# 4. Aplícala
pnpm migration:run
```

`synchronize: false` está fijo en `apps/api/src/config/typeorm.config.ts`. **No se cambia**: pierde datos y rompe el flujo de migraciones. En producción, `docker-entrypoint.sh` corre `migration:run` + seed antes de arrancar la API.

Adminer (UI de la DB): `docker compose --profile tools up -d` → http://localhost:8081

---

## 🧩 Agregar una feature (receta)

1. **Contrato** — `packages/contracts/src/mi-feature.ts`: schemas Zod + tipos, exporta en `index.ts`.
2. **API** — `apps/api/src/mi-feature/`: `*.entity.ts`, `*.service.ts`, `*.controller.ts` (con `ZodValidationPipe` + `JwtAuthGuard`), `*.module.ts` registrado en `app.module.ts`. Genera y corre la migración.
3. **Web** — `apps/web/src/`: service en `services/`, hooks TanStack Query en `hooks/`, página en `pages/`, ruta en `config/navigation.ts` y `router.tsx`.

Lógica de dominio (services, reglas de negocio, cálculo) va con **test primero**. Controladores, UI y glue se testean después, pero se testean.

Componentes shadcn nuevos: `cd apps/web && pnpm dlx shadcn@latest add <componente>`.

> Si agregas una dependencia `workspace:*` a una app, **actualiza su Dockerfile**. Los dos copian `packages/` entero justamente para no volver a romper el deploy, pero el stage de runtime de la API sí lista los `dist` uno por uno.

---

## 🔐 Auth

Se controla **100% con variables de entorno**; el frontend consulta `GET /api/auth/config` y renderiza el login según lo activo.

- `AUTH_LOCAL_ENABLED=true` → email + password. El seed crea el admin.
- `GOOGLE_CLIENT_ID` + `GOOGLE_CLIENT_SECRET` → activa el botón de Google. Con solo `ADMIN_EMAIL` (sin password), ese correo recibe rol admin en su primer login.
- Ambos a la vez: si un usuario local entra luego con Google con el mismo email, las cuentas se vinculan solas.

El JWT viaja en cookie `httpOnly` (`app_session`): el frontend nunca toca el token. CORS exige orígenes exactos porque `credentials: true`.

> Para el demo, el login con Google es el camino: **la PYME no debe ver una seed phrase jamás**. La smart account ERC-4337 se desplegará en segundo plano después del login — está en el diseño, todavía no en el código.

---

## 🚢 Deploy en Coolify

Cada app se despliega como recurso independiente con su Dockerfile (**build context = raíz del repo**):

|             | API                   | Web                                     |
| ----------- | --------------------- | --------------------------------------- |
| Dockerfile  | `apps/api/Dockerfile` | `apps/web/Dockerfile`                   |
| Puerto      | 3000                  | 80                                      |
| Healthcheck | `/health`             | `/`                                     |
| Variables   | runtime (`.env`)      | **build args**: se hornean en el bundle |

`API_PORT` **no se define** en Coolify: el default es 3000 y el healthcheck del Dockerfile está fijado ahí.

El front recibe sus dos variables como **build args**, no como entorno de runtime — cambiarlas exige reconstruir la imagen:

```
VITE_API_URL=https://api.tudominio.com
VITE_EXPLORER_TX_URL=https://sepolia.arbiscan.io/tx
```

> Si añades una variable `VITE_*` al código, **declárala también como `ARG` y `ENV` en `apps/web/Dockerfile`**. Sin eso el build no falla: la app compila y la variable llega como `undefined` a producción. Es el mismo tipo de deriva que ya rompió un deploy con una dependencia `workspace:*`.

Cookies entre dominios:

| Caso                                  | Config                                                    |
| ------------------------------------- | --------------------------------------------------------- |
| `app.dominio.com` + `api.dominio.com` | `COOKIE_DOMAIN=.dominio.com` y `COOKIE_SAMESITE=lax`      |
| Dominios distintos                    | `COOKIE_SAMESITE=none` + `COOKIE_SECURE=true` obligatorio |

Probar los Dockerfiles en local antes de subir:

```bash
docker compose -f docker-compose.prod.yml up -d --build
# api → http://localhost:3000/health · web → http://localhost:8090
docker compose -f docker-compose.prod.yml down
```

---

## 📌 Decisiones técnicas (para no pelearse con ellas)

- **pnpm 11 pineado** (`packageManager` + corepack). `pnpm-workspace.yaml` declara `allowBuilds` — sin eso pnpm bloquea los postinstall nativos (esbuild, tailwind oxide).
- **API en CommonJS** (no NodeNext): imports sin sufijo `.js` y la CLI `typeorm-ts-node-commonjs` funciona sin hacks.
- **bcryptjs** (JS puro): cero bindings nativos en Alpine/CI.
- **Los Dockerfiles copian `packages/` entero** y compilan con `pnpm --filter @app/<app>... build`. Copiarlos uno por uno fue lo que rompió un deploy sin que el CI se enterara.
- **Husky + lint-staged** (prettier automático) y **commitlint**: [conventional commits](https://www.conventionalcommits.org/es) obligatorios.
- Código e identificadores en **inglés**, comentarios y docs en **español**.

---

Base del monorepo: [HeyArnoldo/TemplateFullStack](https://github.com/HeyArnoldo/TemplateFullStack).
