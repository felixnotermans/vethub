# AGENTS.md

## Project overview

VetHub is a veterinary clinic management application (Pet Clinic demo). It exposes a REST API consumed by a SvelteKit frontend. The two halves are independently deployable and share no build tooling.

**Server stack:** Java 25 (virtual threads enabled), Spring Boot, Spring Security (Basic Auth), Spring Data JPA, Hibernate, Liquibase, MapStruct, Lombok, H2 (dev/test), SpringDoc/OpenAPI, Gradle.

**Client stack:** SvelteKit 2 / Svelte 5, TypeScript, Bun, openapi-fetch, Tailwind CSS v4, shadcn-svelte (bits-ui), lucide-svelte, svelte-sonner, tailwind-variants.

---

## Repo structure

Two fully independent sub-projects — no monorepo tooling:

| Directory | Stack |
|-----------|-------|
| `/server` | Spring Boot (Java 25), Gradle wrapper |
| `/client` | SvelteKit 2 / Svelte 5, Bun |

Toolchain versions are pinned in root `mise.toml`. Run `mise install` before anything else.

---

## Server (`/server`)

### Commands

```bash
./gradlew bootRun          # start dev server on :8080
./gradlew test             # run all tests
./gradlew test --tests "dev.ilionx.workshop.api.vet.controller.VetControllerTest"  # single class
./gradlew test --tests "dev.ilionx.workshop.api.vet.controller.VetControllerTest.methodName"  # single method
./gradlew check            # lint (Checkstyle, PMD, SpotBugs, CodeNarc, CPD)
./gradlew spotlessApply    # format code
./gradlew spotlessCheck    # verify formatting
./gradlew bootJar          # build fat JAR
```

### Critical gotchas

- **`spotlessApply` runs automatically on every `JavaCompile` task.** Any `./gradlew build` or `./gradlew compileJava` will silently reformat source files. Expect unexpected git diffs.
- **`-Werror` is enforced.** All compiler warnings are errors. Fix warnings; do not suppress them.
- **No external database.** Dev and test profiles use H2 in-memory. `application-dev.yml` seeds data via Liquibase `contexts: tst`. Production uses `contexts: prd`.

### Architecture

Feature-first packaging by domain (`owner`, `pet`, `vet`, `visit`). Each domain follows `controller → service → repository → model/{mapper,request,response,validator}`. See `.opencode/rules/api-design.md` for full conventions.

### Testing

Extend `UnitTest` (Mockito, no Spring) or `IntegrationTest` (full context, MockMvc). Seeded rows with `id <= 6` (vets/pet-types) and `id <= 3` (specialties) are never deleted — test factories rely on them. See `.opencode/rules/testing.md` for full details.

---

## Client (`/client`)

### Commands

```bash
bun install                # install deps (bun.lock is authoritative, ignore package-lock.json)
bun run dev                # start dev server
bun run build              # production build
bun run check              # svelte-check type-check
bun run sync:api           # full OpenAPI sync (boots server, regenerates types, stops server)
bun run generate:api       # regenerate types from already-running server
```

**There are no client-side tests.**

### OpenAPI codegen

`src/lib/types/api.d.ts` is **generated — never edit it by hand**. After any server API change, run `bun run sync:api` (or `bun run generate:api` if the server is already running). See `.opencode/rules/codegen.md` for the full workflow.

### Quirks

- **Basic Auth** is embedded at client init time (`src/lib/api/client.ts`). SSR context uses `Buffer` instead of `btoa`.
- **`__APP_VERSION__`** is a Vite-injected global defined in `vite.config.ts`. Don't remove it.

---

## Conventions

- Never hand-edit generated files. Never use raw strings for API paths (use `Paths.java`). Always use MapStruct for entity↔DTO mapping.
- Run `./gradlew spotlessApply` before committing Java changes. Match the Spotless import order: `"", "java|jakarta|javax", "groovy", "org", "com", "\\#"`.
- Do not alter Liquibase seed data. Seeded IDs are referenced by test factories.
- See `.opencode/rules/code-style-server.md` and `.opencode/rules/code-style-client.md` for detailed style rules.

---

## Environment

`mise.toml` auto-sets `SERVER_PORT=8080` and `SPRING_PROFILES_ACTIVE=dev` on `cd`.

Optional client overrides in `client/.env`:
```
VITE_SERVER_BASE_URL=http://localhost:8080
VITE_API_USERNAME=user
VITE_API_PASSWORD=password
```

Dev credentials are hardcoded in `application-dev.yml`: `user / password`.
