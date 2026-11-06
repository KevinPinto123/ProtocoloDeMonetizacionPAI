# Visión general de la arquitectura

Este documento es un **recorrido completo** del sistema: qué piezas existen, dónde viven, cómo se hablan entre sí y qué estado tiene cada una hoy. Es el punto de entrada para alguien que nunca vio el repo.

No reemplaza a los otros documentos de `docs/`, los complementa:

| Documento                                                  | Para qué sirve                                                                                                                        |
| ---------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| **Este archivo**                                           | Mapa completo del sistema: módulos, contratos, flujo de datos, estado real                                                            |
| [`arquitectura.md`](arquitectura.md)                       | Las **decisiones que no se vuelven a discutir**: formato de la hoja de Merkle, fórmula del borrowing base, por qué `ChainPort` existe |
| [`referencia-pai-arbitrum.md`](referencia-pai-arbitrum.md) | El caso de uso de negocio, la rúbrica del hackathon, el guion de demo                                                                 |
| [`design/`](design/)                                       | El sistema visual del panel (Nocturne)                                                                                                |
| [`../README.md`](../README.md)                             | Arranque, `.env`, scripts, deploy en Coolify                                                                                          |

> Verificado contra el código en `main` el 2026-08-12. Si algo de acá y el código discrepan, gana el código — actualiza este archivo.

---

## 1. Qué es el proyecto, en una frase

Una PYME sin activos fijos convierte su **cartera de derechos de cobro** (facturas por cobrar) en garantía verificable on-chain, y obtiene financiamiento contra ella — sin que el prestamista tenga que confiar en el operador de la plataforma para saber cuánto vale esa cartera.

**La frontera de confianza:** todo lo que es "un PDF más" vive en Web2 (Postgres + S3). Todo lo que un prestamista necesita **poder verificar sin confiar en nadie** vive on-chain: la huella de la evidencia, quién la certificó, cuánto vale, y quién tiene el dinero.

---

## 2. Mapa del monorepo

```text
ProtocoloDeMonetizacionPAI/
├── apps/
│   ├── api/            NestJS 11 · TypeORM · Postgres · auth JWT en cookie httpOnly
│   └── web/             React 19 · Vite 7 · Tailwind v4 · shadcn/ui · TanStack Query
│
├── packages/            Código compartido, sin servidor propio
│   ├── contracts/        Schemas Zod compartidos api ↔ web (⚠️ NO son smart contracts)
│   ├── merkle/            Hoja canónica del árbol + multiproof + vectores dorados
│   ├── borrowing-base/    Motor de riesgo, especificación normativa en TypeScript
│   ├── evm/               ABIs, tipos, deploy/smoke tooling — generados desde chain/
│   └── tsconfig/          tsconfig.json base compartida
│
├── chain/                Solidity — Foundry
│   ├── src/                7 contratos
│   ├── script/              Deploy.s.sol: despliega y verifica el cableado
│   ├── deployments/         direcciones ya desplegadas por chainId
│   ├── test/                67 tests Foundry
│   └── stylus/              Rust — todavía vacío (motor de valorización on-chain, pendiente)
│
└── docs/                 Este documento y los demás listados arriba
```

El "choque de nombres" a tener presente: **`packages/contracts` son schemas Zod** (validación de datos entre API y web), no tienen nada que ver con los smart contracts. Los smart contracts viven en `chain/`.

---

## 3. La frontera Web2 ↔ Web3

```text
        Web2 (apps/, packages/contracts, merkle, borrowing-base)
        ───────────────────────────────────────────────────────
        expediente · evidencias (S3/MinIO) · certificaciones
        divulgación selectiva · UI · sesión de usuario
                              │
                              │  se encuentran acá:
                              │
              packages/merkle    → hoja canónica + vectores dorados
              ChainPort          → interfaz de dominio + 2 adapters
              packages/evm       → ABIs generados desde chain/
                              │
        ───────────────────────────────────────────────────────
        Web3 (chain/ — Arbitrum Sepolia, chainId 421614)
        AssetRegistry · CertificationAttestor · PAICertificate
        CollateralVault · BorrowingBaseEngine · MockUSDC
```

Reparto aproximado: **~55% Web2, ~45% cadena.** Un PDF no gana nada por vivir on-chain; el valor de la cartera y la custodia del dinero sí lo ganan.

Regla dura, repetida porque todo el sistema depende de ella: **cuando Postgres y la cadena discrepan, gana la cadena.** Postgres es un índice de lo que la cadena ya dijo, no la fuente de verdad.

---

## 4. Backend — `apps/api` (NestJS)

### 4.1 Módulos registrados (`app.module.ts`)

| Módulo               | Responsabilidad                                                                        |
| -------------------- | -------------------------------------------------------------------------------------- |
| `ChainModule`        | Elige el adapter (`in-memory` / `arbitrum`) según `CHAIN_ADAPTER` y expone `ChainPort` |
| `EvidenceModule`     | Sube archivos a S3/MinIO, calcula SHA-256, guarda metadatos                            |
| `AssetsModule`       | Expedientes: registro, snapshot on-chain, intents de registro                          |
| `DisclosureModule`   | Construye el multiproof de divulgación selectiva                                       |
| `UsersModule`        | Usuarios y su rol de negocio                                                           |
| `AuthModule`         | Login local + Google OAuth, JWT en cookie `httpOnly`                                   |
| `HealthModule`       | `/health` para el healthcheck de Docker/Coolify                                        |
| `VerificationModule` | `GET /verify/:code` — verificación pública, sin sesión                                 |

### 4.2 Endpoints por controlador

| Controlador              | Rutas                                                                                                                                                                                   |
| ------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `AssetsController`       | `POST /assets` · `GET /assets/:id` · `GET /assets/:id/chain` · `GET /assets/:id/certification-chain` · `POST /assets/:id/registration-intent` · `POST /assets/:id/confirm-registration` |
| `EvidenceController`     | `GET /evidence` · `POST /evidence`                                                                                                                                                      |
| `DisclosureController`   | `POST /disclosure/:assetId/preview`                                                                                                                                                     |
| `VerificationController` | `GET /verify/:assetId` (pública)                                                                                                                                                        |
| `ChainIntentController`  | `POST /chain/:action` — arma un _intent_ de transacción sin firmarlo                                                                                                                    |
| `AuthController`         | `GET /auth/config` · `POST /auth/register` · `POST /auth/login` · `GET /auth/me` · `POST /auth/logout` · `GET /auth/google` · `GET /auth/google/callback`                               |
| `HealthController`       | `GET /health`                                                                                                                                                                           |

### 4.3 `ChainPort` — la cadena como puerto, no como dependencia

El dominio de la API **no importa Viem, ni direcciones, ni ABIs directamente**. Habla con una interfaz:

```ts
// apps/api/src/chain/chain.port.ts
interface ChainPort {
  registerAsset(input): Promise<TxRef>;
  attest(input): Promise<TxRef>;
  revokeAttestation(input): Promise<TxRef>;
  getAsset(assetId): Promise<OnChainAsset | null>;
  computeBorrowingBase(input): Promise<BorrowingBaseResult>;
}
```

Dos implementaciones, elegidas por `CHAIN_ADAPTER` en el `.env`:

- **`in-memory.adapter.ts`** — guarda estado en memoria, sin cadena real. Es lo que usan dev, tests y la mitad Web2 de la demo. No simula lógica de negocio de forma distinta a la cadena: si el motor Stylus algún día calcula distinto que este adapter, el adapter está mal.
- **`arbitrum.adapter.ts`** — lee/escribe contra Arbitrum Sepolia usando `packages/evm` (Viem). **Ya no es un stub**, está implementado.

**La API nunca firma transacciones de valor.** `ChainIntentController` arma el _intent_ (los parámetros de la transacción) y el navegador de quien tiene la wallet la firma vía EIP-1193. No hay ninguna `ATTESTOR_PRIVATE_KEY` ni equivalente en el backend — si algo la necesita para mover fondos, es un bug de arquitectura.

### 4.4 Persistencia

TypeORM contra Postgres, con `synchronize: false` fijo (`apps/api/src/config/typeorm.config.ts`). El esquema **solo** cambia con migraciones (`apps/api/src/database/migrations`). El seed (`apps/api/src/database/seeds`) crea el admin inicial y es idempotente.

### 4.5 Auth y roles de negocio

El rol de cada usuario **no se guarda a mano**: se deriva de en qué lista de correos aparece, vía variables de entorno (`apps/api/src/auth/role-mapping.ts`):

```text
ADMIN_EMAIL       → UserRole.ADMIN
CERTIFIER_EMAILS  → UserRole.CERTIFIER   (lista separada por comas)
FUND_EMAILS       → UserRole.FUND        (lista separada por comas)
cualquier otro     → UserRole.PYME       (default)
```

El JWT viaja en una cookie `httpOnly` (`app_session`); el frontend nunca toca el token directamente. Login local (email + password) y Google OAuth pueden convivir — si un usuario local entra después con Google usando el mismo correo, las cuentas se vinculan solas.

---

## 5. Frontend — `apps/web` (React + Vite)

### 5.1 Rutas (`router.tsx`)

| Ruta                       | Página                                              | Protegida |
| -------------------------- | --------------------------------------------------- | --------- |
| `/`                        | Landing pública                                     | No        |
| `/login`, `/register`      | Auth                                                | No        |
| `/verify`, `/verify/:code` | Verificación pública de un expediente               | No        |
| `/panel`                   | `overview.tsx` — dashboard                          | Sí        |
| `/expediente`              | `dossier.tsx`                                       | Sí        |
| `/evidencias`              | `evidence.tsx`                                      | Sí        |
| `/divulgacion`             | `disclosure.tsx` — **el multiproof en vivo**        | Sí        |
| `/borrowing-base`          | `borrowing-base.tsx` — **el desglose recomputable** | Sí        |
| `/certificacion`           | `certification.tsx`                                 | Sí        |
| `/prestamo`                | `loan.tsx`                                          | Sí        |
| `/historial`               | `credit-history.tsx`                                | Sí        |
| `/actividad`               | `activity.tsx`                                      | Sí        |

Las páginas son lazy-loaded (un chunk por ruta). `/disclosure` redirige a `/divulgacion` por compatibilidad con enlaces viejos.

### 5.2 Capas dentro de `apps/web/src`

| Carpeta                             | Qué hay                                                                                                                                                            |
| ----------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `domain/`                           | Lógica pura y testeada aparte de la UI: `borrowing-base.ts`, `disclosure-selection.ts`, `dossier-view.ts`, `loan-intent.ts`, `certification-intent.ts`, `money.ts` |
| `services/`                         | Llamadas HTTP a la API: `auth.api.ts`, `disclosure.api.ts`, `evidence.api.ts`, `verification.api.ts`, `transaction-intent.ts`                                      |
| `hooks/`                            | TanStack Query sobre los services: `use-auth`, `use-disclosure`, `use-evidence`, `use-transaction-intent`, `use-wallet-submitter`                                  |
| `context/`                          | `wallet-provider.tsx` (conexión MetaMask global), `disclosure-selection-provider.tsx`                                                                              |
| `components/`, `layouts/`, `pages/` | UI (shadcn/ui sobre tokens Nocturne)                                                                                                                               |

### 5.3 Wallet / MetaMask

`context/wallet-provider.tsx` mantiene la conexión de MetaMask **a nivel de toda la app** (no por página). El flujo es:

1. La API arma un _intent_ de transacción (`POST /chain/:action`) — solo datos, sin firmar.
2. `use-wallet-submitter.ts` lo pasa a la wallet conectada del navegador vía EIP-1193.
3. El usuario firma en MetaMask. La transacción sale de **su** wallet, nunca del backend.

Esto es lo que sostiene el argumento legal del proyecto: el dinero (USDC) nunca toca un servidor que la plataforma controle.

---

## 6. Cadena — `chain/` (Solidity, Foundry)

### 6.1 Los 7 contratos

| Contrato                    | Rol                                                                                                                             |
| --------------------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| `ReceivableLeaf.sol`        | Define la hoja canónica del árbol de Merkle (no despliega estado, es la codificación compartida)                                |
| `AssetRegistry.sol`         | Guarda el `merkleRoot` de cada expediente y su estado. Nunca sube el archivo ni datos personales — solo la huella               |
| `CertificationAttestor.sol` | Cada certificador firma **desde su propia wallet**; ninguno ve el expediente completo del otro                                  |
| `PAICertificate.sol`        | Token **soulbound** (no transferible): certifica que un expediente cumplió el gate de certificación, no que alguien es su dueño |
| `BorrowingBaseEngine.sol`   | Recomputa el borrowing base en Solidity, determinista, sobre cuotas ya divulgadas — sin oráculos                                |
| `CollateralVault.sol`       | Ciclo de vida del préstamo garantizado. El USDC va del fondo al vault y del vault a la PYME en la misma transacción             |
| `MockUSDC.sol`              | ERC-20 de 6 decimales para la demo — **nunca desplegar en producción**                                                          |

### 6.2 Estado del deploy — esto ya pasó

A diferencia de lo que dice el README en su tabla de estado (que lo marca 🟡 pendiente), **los 7 contratos ya están desplegados en Arbitrum Sepolia** (`chainId 421614`), registrados en `chain/deployments/421614.json`:

| Contrato                | Dirección                                    |
| ----------------------- | -------------------------------------------- |
| `AssetRegistry`         | `0xFda73e63797dcD7940e1Ce0B000dbE17d03c3208` |
| `CertificationAttestor` | `0xCc4CF2d2761cFf9a2c871A2bA8e5BCbb6CEB44F6` |
| `PAICertificate`        | `0x6b15e6715ABdD6D01661bAcf056193D6f9f33FE8` |
| `BorrowingBaseEngine`   | `0x3Ea2f1541d2F8aB3137d2A4F64067E3d69BaA10B` |
| `CollateralVault`       | `0xa281d5342327D35068FCcB9E99f6c0F57ee59611` |
| `MockUSDC`              | `0x062293daDabE79FC0A47665126B71070cD5689C8` |

> Bloque de deploy: `296546459`. Ese número es el que va en `CHAIN_DEPLOYMENT_BLOCK` — sin él, el indexador escanearía la cadena desde el bloque 0.

**Pero tu `.env` local sigue en `CHAIN_ADAPTER=in-memory`** con las direcciones vacías. Es decir: el deploy existe, pero para que tu instancia local hable con la cadena real hace falta copiar estas direcciones al `.env` (ver [§9, Variables de entorno](#9-variables-de-entorno--el-mapa-corto)). Mientras tanto la API funciona entera sin cadena — es el modo por defecto para dev.

### 6.3 Lo que falta

- **`chain/stylus/`** está vacío. El `BorrowingBaseEngine` hoy es Solidity, no Rust — la fórmula ya está especificada en `packages/borrowing-base` con vectores dorados, lista para que el motor Stylus la reproduzca byte a byte cuando exista.
- **Cuenta inteligente ERC-4337**: no existe. El login con Google funciona; la smart account en segundo plano detrás de ese login, no.

---

## 7. Packages compartidos

| Package                   | Qué encapsula                                                                                                                                                   | Consumido por                                                                                                                                         |
| ------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| **`@app/merkle`**         | Codificación de la hoja, construcción del árbol (envuelve `@openzeppelin/merkle-tree`, no lo reimplementa), multiproof, vectores dorados                        | API (arma proofs), web (verifica en el navegador), tests de Foundry (mismos vectores)                                                                 |
| **`@app/borrowing-base`** | La fórmula de riesgo, en enteros, con vectores dorados                                                                                                          | Web (`/borrowing-base`, hoy corre en el navegador como "cálculo local de referencia"), y es la especificación que `BorrowingBaseEngine.sol` reproduce |
| **`@app/contracts`**      | Schemas Zod: `assets.ts`, `auth.ts`, `disclosure.ts`, `verification.ts`, `common.ts` — el contrato de datos entre API y web                                     | API (`ZodValidationPipe`) y web (tipos + parseo de respuestas)                                                                                        |
| **`@app/evm`**            | ABIs y tipos **generados** desde `chain/` (nunca a mano), direcciones de deploy (`deployments.ts`), _intents_ de transacción, herramientas de deploy/smoke test | `arbitrum.adapter.ts` en la API                                                                                                                       |

La regla que evita que estos dos lados se desincronicen: `packages/merkle/fixtures/golden-vectors.json` y los vectores de `packages/borrowing-base/fixtures/` son leídos **tanto** por los tests de TypeScript **como** por los tests de Foundry (`forge test --match-contract GoldenVectors`). Si un lado cambia la codificación sin avisar al otro, el CI se pone rojo — no hay forma de que el desacuerdo llegue silencioso hasta la demo.

---

## 8. Flujo de datos de punta a punta

Ejemplo concreto, del alta de un expediente al préstamo fondeado:

```text
1. PYME sube evidencias (PDFs de contratos de cobro)
   → EvidenceModule calcula SHA-256, sube a S3/MinIO, guarda metadatos en Postgres

2. Se arma el expediente: cada cuota se codifica como hoja canónica
   (@app/merkle) y se construye el árbol → merkleRoot

3. Registro on-chain: AssetsController pide un intent de registro,
   la PYME lo firma con su wallet → AssetRegistry.sol guarda el merkleRoot

4. Certificación: 2-3 certificadores (contador, abogado, auditor) firman
   atestaciones EIP-712 sobre el expediente, cada uno desde su propia
   wallet → CertificationAttestor.sol. Al completar el gate, se emite
   el PAICertificate (soulbound)

5. Divulgación selectiva: el fondo pide ver la cartera. La PYME elige
   qué cuotas mostrar → DisclosureModule construye un multiproof real.
   El merkleRoot NO cambia con la selección — esa es la prueba de que
   nada se está inventando

6. Borrowing base: se recomputa el valor prestable a partir de las
   cuotas divulgadas (descuento por plazo, haircut de mora,
   concentración, continuidad) → BorrowingBaseEngine.sol, o su
   espejo en @app/borrowing-base mientras Stylus no exista

7. Préstamo: el fondo transfiere USDC directo al CollateralVault,
   que lo libera a la PYME — en la misma transacción, sin pasar
   por el servidor
```

Los dos momentos que un jurado ve y no vio en otras 40 demos son el **paso 5** (`/divulgacion`) y el **paso 6** (`/borrowing-base`): en ambos, lo que se muestra se puede recomputar, no hay que creerle a la pantalla.

---

## 9. Variables de entorno — el mapa corto

La fuente autoritativa completa es `apps/api/src/config/env.validation.ts` (Zod, se valida una sola vez al arrancar) y el `.env.example` de la raíz. Resumen de las categorías:

| Grupo           | Ejemplos                                                                                          | Siempre obligatorio                       |
| --------------- | ------------------------------------------------------------------------------------------------- | ----------------------------------------- |
| Postgres        | `DB_HOST`, `DB_PORT`, `DB_USER`, `DB_PASSWORD`, `DB_NAME`                                         | Sí                                        |
| Sesión          | `JWT_SECRET` (≥16 chars)                                                                          | Sí                                        |
| Storage         | `STORAGE_REGION`, `STORAGE_BUCKET`, `STORAGE_ACCESS_KEY`, `STORAGE_SECRET_KEY`                    | Sí                                        |
| Cadena          | `CHAIN_ADAPTER`, `CHAIN_RPC_URL`, `CHAIN_DEPLOYMENT_BLOCK`, `*_ADDRESS` (6 direcciones, ver §6.2) | Solo si `CHAIN_ADAPTER=arbitrum`          |
| Roles de demo   | `CERTIFIER_EMAILS`, `FUND_EMAILS`                                                                 | No, pero sin ellas todos entran como PYME |
| Build del front | `VITE_API_URL`, `VITE_EXPLORER_TX_URL`                                                            | Se hornean en el bundle, no son runtime   |

> Detalle completo, con cada variable explicada, en el README bajo [`🔑 El .env completo`](../README.md#-el-env-completo).

---

## 10. Infraestructura

### 10.1 Local

`docker-compose.yml` levanta **tres** servicios independientes — ninguno arranca solo con `pnpm db:up` salvo Postgres:

| Servicio                          | Comando para levantarlo                 | Puerto                           |
| --------------------------------- | --------------------------------------- | -------------------------------- |
| `db` (Postgres 16)                | `pnpm db:up`                            | `5432` (o el que fije `DB_PORT`) |
| `minio`                           | `docker compose up -d minio minio-init` | `9000` (API), `9001` (consola)   |
| `adminer` (opcional, UI de la DB) | `docker compose --profile tools up -d`  | `8081`                           |

### 10.2 Producción — Coolify

Cada app se despliega como recurso independiente, mismo repo como build context:

|            | API                   | Web                                                     |
| ---------- | --------------------- | ------------------------------------------------------- |
| Dockerfile | `apps/api/Dockerfile` | `apps/web/Dockerfile`                                   |
| Puerto     | `3000`                | `80`                                                    |
| Variables  | runtime (`.env`)      | **build args** — cambiarlas exige reconstruir la imagen |

`docker-compose.prod.yml` sirve para probar ambos Dockerfiles en local antes de subir.

### 10.3 CI (`.github/workflows/ci.yml`)

Cuatro jobs, en paralelo:

| Job         | Qué corre                                                                                           |
| ----------- | --------------------------------------------------------------------------------------------------- |
| `ci`        | lint · typecheck · build · tests de todos los workspaces Node                                       |
| `e2e`       | Playwright contra el panel (mide tokens de diseño computados, no píxeles)                           |
| `docker`    | Construye `apps/api/Dockerfile` y `apps/web/Dockerfile` — las mismas imágenes que despliega Coolify |
| `contracts` | `forge fmt --check` · `forge build --sizes` · `forge test -vvv` sobre `chain/`                      |

El job `docker` existe porque el job `ci` compila sobre el checkout completo del monorepo y nunca ejerce el subconjunto de archivos que Docker sí ve — un CI verde llegó a convivir con un deploy roto (`Cannot find module '@app/merkle'`). Ahora no puede pasar sin que el job `docker` lo note.

---

## 11. Estado real (resumen, ver también el README)

| Pieza                                       | Estado                                                             |
| ------------------------------------------- | ------------------------------------------------------------------ |
| Hoja canónica de Merkle + multiproof        | ✅ Funciona, con vectores dorados verificados en TS y Solidity     |
| Motor de borrowing base (TypeScript)        | ✅ Funciona                                                        |
| Panel web (13 páginas/rutas)                | ✅ Funciona, responsive                                            |
| 7 contratos Solidity                        | ✅ Escritos, testeados, **y ya desplegados en Sepolia** (`421614`) |
| `packages/evm` (ABIs + tooling de deploy)   | ✅ Funciona                                                        |
| Adaptador `arbitrum.adapter.ts`             | ✅ Implementado (no es un stub)                                    |
| Conexión MetaMask global + firma de intents | ✅ Funciona                                                        |
| `.env` local apuntando a la cadena real     | 🟡 Falta configurar — hoy corre en `CHAIN_ADAPTER=in-memory`       |
| Motor de valorización en Stylus (Rust)      | 🔴 No existe — `chain/stylus/` vacío                               |
| Cuenta inteligente ERC-4337                 | 🔴 No existe                                                       |

---

## 12. Por dónde seguir

- ¿Vas a tocar la hoja de Merkle o el borrowing base? Lee primero [`arquitectura.md`](arquitectura.md) — ahí están las decisiones que no se negocian.
- ¿Vas a agregar una pantalla o un endpoint nuevo? La receta paso a paso está en el README, sección [`🧩 Agregar una feature`](../README.md#-agregar-una-feature-receta).
- ¿Vas a desplegar o cambiar el `.env`? README, sección [`🚢 Deploy en Coolify`](../README.md#-deploy-en-coolify).
