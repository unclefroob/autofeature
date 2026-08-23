---
name: backend-stack-profiles
purpose: Detect the backend's stack and map the four standard layers onto it. Read by /autofeature:backend-api-skills before any audit or code generation.
status: CUSTOM
---

# Backend Stack Profiles

`api-standard.md` states the law in layer names. This file turns those names into the concrete files, imports and commands of the repo actually in front of you.

**Detect, never assume.** The single largest failure mode of a portable standard is generating Express idioms into a Fastify repo. Run detection first, state the detected profile out loud, and stop if it comes back ambiguous.

---

## Step 1: Detect

```bash
ROOT="${1:-$(pwd)}"
cd "$ROOT" || exit 1

# Language / runtime
ls package.json tsconfig.json go.mod pyproject.toml requirements.txt Cargo.toml 2>/dev/null

# Node: framework, data layer, validator, test runner — dependencies only
node -e '
  const p = require("./package.json");
  const d = {...p.dependencies, ...p.devDependencies};
  const has = (...k) => k.filter(x => d[x]).map(x => `${x}@${d[x]}`);
  console.log("framework:", has("express","fastify","@nestjs/core","koa","hapi","hono","@hapi/hapi").join(" ") || "?");
  console.log("data:     ", has("mongoose","prisma","@prisma/client","typeorm","sequelize","knex","drizzle-orm","pg","mysql2").join(" ") || "?");
  console.log("validate: ", has("zod","joi","yup","class-validator","express-validator","@sinclair/typebox","valibot").join(" ") || "?");
  console.log("test:     ", has("jest","vitest","mocha","tap","node:test","supertest","mongodb-memory-server","testcontainers").join(" ") || "?");
  console.log("scripts:  ", JSON.stringify(p.scripts || {}));
' 2>/dev/null

# Existing layering — count, do not read
for d in routes controllers services models repositories handlers modules domain features api src/routes src/controllers src/services src/models; do
  [ -d "$d" ] && echo "$d: $(find "$d" -type f \( -name '*.ts' -o -name '*.js' \) | wc -l) files"
done
```

Then record a one-line profile and **say it back to the user** before doing anything else:

> Detected: Node 20 · Express 4 · TypeScript (strict) · Mongoose 8 · zod · Jest + supertest · layering `src/{routes,controllers,services,models}` present.

**Ambiguity stops the run.** Two frameworks in `dependencies`, no recognisable data layer, or no test runner at all → report what you found and ask, rather than picking. A wrong profile produces confidently wrong code in every file it touches.

---

## Step 2: Map the layers

The law is fixed; the file that satisfies it is not.

| Layer | Express | Fastify | NestJS | Koa/Hono |
|-------|---------|---------|--------|----------|
| **Route** | `Router()` + `app.use('/api', r)` | plugin with `fastify.route(…)` | `@Controller()` decorators + module | router middleware |
| **Controller** | `async (req,res,next)` handler module | route `handler` fn | the `@Controller` class methods | ctx handler |
| **Service** | `services/<domain>Service.ts` function module | same | `@Injectable()` provider *(a class here — Nest's DI is the exception to the function-module rule; it must still import nothing HTTP-shaped)* | same |
| **Model** | Mongoose schema | Mongoose schema | entity / schema | schema |
| **Error funnel** | `errorHandler` registered LAST | `setErrorHandler` | exception filter | error middleware |
| **Validation** | inline zod `.parse` in controller | schema on the route def, or zod in handler | DTO + `class-validator` pipe | zod in handler |

| Layer | Mongoose | Prisma | TypeORM | Knex/Drizzle |
|-------|----------|--------|---------|--------------|
| **Model file** | `models/Foo.ts` schema + model | `schema.prisma` + generated client | `entities/Foo.ts` | `schema.ts` / migrations |
| **Query lives in** | service | service (`prisma.foo.*`) | service (repo injected) | service |
| **Audit tell (V1)** | `.find(` `.aggregate(` `.create(` in a controller | `prisma.` in a controller | `getRepository(` / `.manager.` in a controller | `db(` / `knex(` / raw SQL in a controller |
| **Unit-test DB** | `mongodb-memory-server` | test schema or testcontainers | sqlite/testcontainers | testcontainers |

**Nest note.** NestJS already enforces most of this structurally; on a Nest repo the audit's useful findings collapse to V1 (logic in the controller class), V3 (scoped auth in a guard), V5, V9 and V11. Don't report ceremony Nest already gives you as a win.

---

## Step 3: Discover the commands

Never invent a command. Read them, in this order:

1. `package.json` `scripts` — prefer `typecheck`, `test`, `build`, `lint` by exact name.
2. The CI workflow (`.github/workflows/*.yml`) — **this is the authority on what must pass**, because it is what actually blocks a merge. If CI runs something `scripts` doesn't, CI wins.
3. The repo README's contributing section.

Record the four gate commands. If `typecheck` or `build` genuinely does not exist (plain JS repo), say so and drop that gate rather than substituting a command you guessed.

---

## Non-Node backends

The law is language-neutral and the audit's 🔴 rows (V1, V2, V3, V4, V11) transfer directly. The concrete guidance does not. On Python/Go/Rust:

- Map the layers to the local idiom — FastAPI router → endpoint fn → service module → SQLAlchemy model; Go `handler` → `service` → `repository` → `model`.
- Keep V1/V3/V4/V11 as the audit spine; the framework-specific tells in the tables above do not apply, so derive them from the detected data layer's query surface.
- **Be explicit that guidance is thinner here**, and prefer auditing over generating. `create` mode on a non-Node stack should scaffold from the repo's own existing shape, not from this file's reference implementation.

---

## Repo canon overrides sampling

If `.autofeature/patterns.md` exists (from `/autofeature:patterns`), read it before sampling any code and treat its **Canonical** entries as binding. In a drifted repo the majority pattern is often the deprecated one, and a majority does not out-vote the canon.

Order of authority, highest first:

1. `.autofeature/patterns.md` — the repo's own decided canon
2. `api-standard.md` — the layering law
3. What the repo's existing files happen to do
