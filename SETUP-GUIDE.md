# Setup Guide: FE source với Claude Code

Tài liệu này ghi lại **từng bước dựng source từ đầu**, mỗi phase tương ứng một mốc (branch/tag) trong git. Checkout về mốc nào thì đọc phase đó để hiểu trạng thái source lúc ấy và lý do đằng sau.

| Phase | Nội dung                                                            | Tag gợi ý                  | Trạng thái |
| ----- | ------------------------------------------------------------------- | -------------------------- | ---------- |
| 1     | Init repo + guardrails                                              | `v0.1-init-guardrails`     | Hoàn thành |
| 2     | Cấu trúc thư mục + CLAUDE.md                                        | `v0.2-structure-claude-md` | Hoàn thành |
| 3     | Design tokens + Button + skill `new-component`                      | `v0.3-design-system`       | Chưa làm   |
| 4     | Testing + skill `write-tests` + agent `test-writer`                 | `v0.4-testing`             | Chưa làm   |
| 5     | Hooks + permissions                                                 | `v0.5-hooks`               | Chưa làm   |
| 6     | Feature đầu tiên + agent `frontend-architect` + skill `new-feature` | `v0.6-first-feature`       | Chưa làm   |
| 7     | Lớp review độc lập                                                  | `v0.7-review-layer`        | Chưa làm   |
| 8     | MCP                                                                 | `v0.8-mcp`                 | Chưa làm   |
| 9     | CI + PR workflow                                                    | `v0.9-ci`                  | Chưa làm   |
| 10    | Chuẩn hóa cho team                                                  | `v1.0-team-ready`          | Chưa làm   |

---

## Nguyên tắc xuyên suốt

> Không build skill/subagent trước khi có mẫu chuẩn để chúng bắt chước hoặc kiểm tra.

Thứ tự: làm tay một lần cho đúng, trích thành skill, thêm subagent khi thấy đau.
Mỗi phase là một branch và một PR. Cuối phase tự hỏi: _"Claude đã sai ở đâu? Sửa bằng CLAUDE.md, skill, subagent hay hook?"_

---

## Phase 1: Init repo và guardrails

### Mục tiêu

Repo có tín hiệu lỗi tự động. Claude Code sửa code dựa trên lỗi nhận được (type error, lint error, commit bị chặn). Repo không có tín hiệu thì Claude viết xong là tưởng đúng.

### Stack

Vite, React, TypeScript, pnpm, ESLint, Prettier, Husky, lint-staged.

### Vì sao pnpm

Nhanh hơn, tiết kiệm đĩa (store dùng chung), và chặn _phantom dependencies_: import một package chưa khai báo trong `package.json` sẽ lỗi ngay thay vì "tình cờ chạy được". Claude đôi khi import thư viện nó nhớ là có sẵn, nên lỗi lộ sớm là điều tốt. Dùng npm hoặc yarn vẫn được, miễn cả team thống nhất và đổi lệnh tương ứng.

### Các bước

**1. Tạo project**

```bash
pnpm create vite@latest react-claude-code --template react-ts
cd react-claude-code
git init
pnpm install
```

`git init` phải chạy trong cùng thư mục với `package.json` (xem Troubleshooting #5).

**2. Bật TypeScript strict**

Trong `tsconfig.app.json`, thêm vào `compilerOptions` (template Vite có thể chưa có sẵn):

```jsonc
{
  "compilerOptions": {
    // ...giữ nguyên các option hiện có...
    "strict": true,
    "noUncheckedIndexedAccess": true,
  },
}
```

- `strict`: bật nhóm kiểm tra chặt (null check, implicit any). Là nền tảng để quy ước "không dùng `any`" có ý nghĩa.
- `noUncheckedIndexedAccess`: `arr[0]` có kiểu `T | undefined`, bắt lỗi truy cập phần tử không tồn tại ngay lúc typecheck.

**3. Cài công cụ kiểm tra**

```bash
pnpm add -D husky lint-staged prettier eslint-config-prettier
```

Xác nhận đã cài (phải thấy 3 package kèm phiên bản):

```bash
pnpm list husky lint-staged prettier
```

**4. Thêm scripts vào `package.json`**

```json
{
  "scripts": {
    "dev": "vite",
    "build": "tsc -b && vite build",
    "typecheck": "tsc -b --noEmit",
    "lint": "eslint .",
    "format": "prettier --write .",
    "preview": "vite preview",
    "prepare": "husky"
  }
}
```

Template Vite chỉ tạo sẵn `dev`, `build`, `lint`, `preview`. `typecheck` và `format` phải tự thêm. `prepare` giúp đồng nghiệp chạy `pnpm install` là tự bật husky.

**5. Cấu hình Prettier**

`.prettierrc`:

```json
{
  "semi": true,
  "singleQuote": true,
  "trailingComma": "all",
  "printWidth": 100
}
```

`.prettierignore`:

```
dist
node_modules
pnpm-lock.yaml
```

**6. Nối Prettier với ESLint**

Trong `eslint.config.js`, thêm `eslint-config-prettier` làm phần tử **cuối cùng** để eslint không xung đột với prettier về format:

```js
import prettier from 'eslint-config-prettier';

export default [
  // ...config hiện có...
  prettier, // đặt cuối cùng
];
```

Nếu file dùng `tseslint.config(...)`, thêm `prettier` làm phần tử cuối trong lời gọi đó.

**7. Cấu hình lint-staged**

Thêm vào `package.json`:

```json
{
  "lint-staged": {
    "*.{ts,tsx}": ["eslint --fix", "prettier --write"],
    "*.{json,md,css}": ["prettier --write"]
  }
}
```

**8. Khởi tạo Husky và gắn hook**

```bash
pnpm exec husky init
```

Mở `.husky/pre-commit`, thay toàn bộ nội dung bằng:

```
pnpm lint-staged
pnpm typecheck
```

Mỗi lần commit sẽ format và lint file đã sửa, rồi typecheck toàn dự án. Nếu dự án lớn làm commit chậm, có thể bỏ `typecheck` khỏi hook và để CI đảm nhiệm.

### Cách hook hoạt động

```
git commit
   │
   ▼
git đọc core.hooksPath ──► .husky/_ ──► chạy .husky/pre-commit
                                          ├─ pnpm lint-staged
                                          └─ pnpm typecheck
```

Mặc định git chỉ tìm hook trong `.git/hooks/` (không được đưa vào repo). Husky ghi cấu hình `core.hooksPath` để git tìm trong `.husky/_`. Thư mục `_` do husky tự sinh, không tự tạo tay.

### Checkpoint

```bash
pnpm typecheck && pnpm lint && pnpm build
```

Phải chạy sạch. Sau đó kiểm tra hai thứ:

**a) TypeScript strict hoạt động.** Tạo file tạm `src/test-ts.ts`:

```ts
const items = [1, 2, 3];
const first: number = items[0]; // phải báo lỗi: number | undefined
```

`pnpm typecheck` phải báo lỗi ở dòng thứ hai. Kiểm tra xong thì xóa file.

**b) Husky chặn được commit lỗi.** Cố ý đưa vào lỗi cú pháp, ví dụ `<section id="center"'>`:

```bash
git add .
git commit -m "test hook"
```

Commit phải bị chặn, terminal in ra các dòng như `[STARTED] Running tasks for staged files...`. Sửa lỗi xong rồi commit thật:

```bash
git add .
git commit -m "chore: init project with guardrails"
git tag v0.1-init-guardrails
```

### Ba lớp bảo vệ sau phase này

| Lớp                 | Bắt lỗi gì                        | Khi nào              |
| ------------------- | --------------------------------- | -------------------- |
| TypeScript strict   | Sai kiểu, thiếu null check        | `pnpm typecheck`     |
| ESLint              | Code smell, lỗi logic phổ biến    | `pnpm lint`          |
| Husky + lint-staged | Chặn commit nếu hai lớp trên fail | Mỗi lần `git commit` |

### Skill/Subagent ở phase này

Chưa có. Chưa có quy trình nào đủ ổn định để đóng gói.

### Troubleshooting (các lỗi đã gặp thực tế)

| #   | Triệu chứng                                                | Nguyên nhân                                              | Cách xử lý                                                                                                |
| --- | ---------------------------------------------------------- | -------------------------------------------------------- | --------------------------------------------------------------------------------------------------------- |
| 1   | `typecheck not found`                                      | Script chưa có trong `package.json`                      | Thêm `"typecheck": "tsc -b --noEmit"` vào `scripts`                                                       |
| 2   | `tsc -b --noEmit` báo lỗi lạ về `composite`/`noEmit`       | Project references của Vite                              | Đổi thành `"typecheck": "tsc -p tsconfig.app.json --noEmit"`                                              |
| 3   | `husky not found`                                          | Husky chưa được cài (`pnpm list` trả về `0 packages`)    | Chạy `pnpm add -D husky lint-staged prettier eslint-config-prettier`                                      |
| 4   | Commit file lỗi vẫn thành công, không thấy log lint-staged | Hook không chạy: husky chưa init, `core.hooksPath` trống | `pnpm exec husky init`, kiểm tra `git config core.hooksPath` phải ra `.husky/_`                           |
| 5   | `.git can't be found` khi chạy husky                       | `.git` không cùng thư mục với `package.json`             | Chạy `git init` trong thư mục có `package.json`. Kiểm tra `git rev-parse --show-toplevel` trùng với `pwd` |
| 6   | Không thấy thư mục `.husky/_`                              | Husky chưa chạy thành công                               | Chạy `pnpm exec husky init`; thư mục `_` do husky tự sinh                                                 |
| 7   | `lint-staged` báo "No valid configuration found"           | Block `lint-staged` đặt sai chỗ trong `package.json`     | Đặt ở cấp ngoài cùng, không nằm trong `scripts`                                                           |

### Lệnh chẩn đoán nhanh khi hook không chạy

```bash
git config core.hooksPath      # phải ra .husky/_
cat .husky/pre-commit          # phải có pnpm lint-staged và pnpm typecheck
git rev-parse --show-toplevel  # phải trùng với thư mục chứa package.json
pnpm lint-staged               # chạy tay sau git add để xem có báo lỗi không
```

---

## Phase 2: Cấu trúc thư mục và CLAUDE.md

### Mục tiêu

Claude biết code đặt ở đâu và luật chơi là gì. `CLAUDE.md` được nạp vào mọi session, là nơi duy nhất chắc chắn Claude đọc.

### Các bước

**1. Tạo branch**

```bash
git checkout main
git checkout -b phase/02-structure-claude-md
```

**2. Tạo cấu trúc thư mục** (thêm `.gitkeep` vì git không theo dõi thư mục rỗng)

```bash
mkdir -p src/app src/features src/shared/ui src/shared/lib src/shared/hooks src/tokens
touch src/app/.gitkeep src/features/.gitkeep src/shared/ui/.gitkeep \
      src/shared/lib/.gitkeep src/shared/hooks/.gitkeep src/tokens/.gitkeep
```

| Thư mục                      | Chứa gì                                                            |
| ---------------------------- | ------------------------------------------------------------------ |
| `app/`                       | Providers, router, entry                                           |
| `features/`                  | Mỗi feature một thư mục: `components/`, `hooks/`, `api/`, `model/` |
| `shared/ui/`                 | Design system dùng chung                                           |
| `shared/lib`, `shared/hooks` | Hàm tiện ích, hook dùng chung                                      |
| `tokens/`                    | Design tokens (Phase 3)                                            |

**3. Sinh bản nháp CLAUDE.md:** chạy `claude`, gõ `/init`.

**4. Cắt gọt.** Với mỗi dòng của bản nháp, hỏi: _"Nếu xóa dòng này, Claude có làm sai không?"_

- Không → xóa. Có → giữ. Không chắc → xóa rồi xem Claude có sai không.
- **Xóa:** thứ Claude tự suy ra từ code (tên framework, file entry), thông lệ chung chung ("viết code sạch"), mô tả chi tiết từng file, thứ chưa tồn tại, hướng dẫn dài theo tình huống (thuộc về skill), nguyên tắc dành cho người làm chứ không dành cho Claude.
- **Giữ:** lệnh chạy và cách verify, luật kiến trúc không lộ ra từ code, lựa chọn khác mặc định, điều cấm.
- Viết thành câu mệnh lệnh ngắn, kiểm chứng được ("Không dùng `any`" thay vì "cố gắng tránh `any`").
- Chỉ ghi những gì **đang tồn tại**. Chưa cài Tailwind/Vitest thì không ghi, nếu không Claude sẽ tự cài hoặc báo lỗi.
- Không dùng cú pháp `@docs/...` để import file dài vào CLAUDE.md, vì nó nạp cả file vào mọi session. Dùng link thường.

Các phần chính sau khi cắt gọt (khoảng 40 dòng):

```markdown
# CLAUDE.md

Vite + React 19 + TypeScript app, built in phases as a learning project.
Each phase is documented in [docs/SETUP-GUIDE.md](docs/SETUP-GUIDE.md) (Vietnamese).
When finishing a phase, update its section in SETUP-GUIDE.md in the same commit as the code.

## Commands

Use **pnpm** only. Importing a package missing from `package.json` fails by design:
add the dependency, don't work around it.

- `pnpm dev` | `pnpm typecheck` | `pnpm lint` | `pnpm format` | `pnpm build`
- Done = `pnpm typecheck && pnpm lint && pnpm build` all clean. No test framework yet.

## Architecture

- `src/app`: providers, router.
- `src/features/<name>/{components,hooks,api,model}`: code grouped by feature.
- `src/shared/ui`: design system. `src/shared/{lib,hooks}`: shared utilities.
- `src/tokens`: design tokens.
- NEVER import from `features` into `shared`.

## Conventions

- Function components, named exports. No `any`, no `@ts-ignore`.
- `noUncheckedIndexedAccess` is on: `arr[0]` is `T | undefined`, narrow it.
- Do not add libraries without asking.

## Guardrails

- Pre-commit runs `pnpm lint-staged` then `pnpm typecheck`.
  If a commit is blocked, fix the errors. Never use `--no-verify`.
- In `eslint.config.js`, `eslint-config-prettier` must stay the LAST entry.

## Git

- Branch: `<type>/<short-description>`, type is one of
  phase | feat | fix | chore | docs | explore. Lowercase, hyphens, ASCII only.
- Commits: Conventional Commits (`feat:`, `fix:`, `chore:`, `docs:`, `refactor:`, `test:`).
- One purpose per branch, small PRs. No commits directly to `main`. No `git push --force`.

## Don'ts

- Do not edit `.env*`. Do not change dependencies without asking.
```

**5. Commit**

```bash
git add .
git commit -m "docs: add project structure and CLAUDE.md"
```

### Quy ước git (nằm trong CLAUDE.md)

| Tiền tố branch | Dùng cho                      | Ví dụ                     |
| -------------- | ----------------------------- | ------------------------- |
| `phase/`       | Một phase trong roadmap       | `phase/03-design-system`  |
| `feat/`        | Feature thật                  | `feat/product-list`       |
| `fix/`         | Sửa lỗi                       | `fix/button-focus-ring`   |
| `chore/`       | Cấu hình, dependency, tooling | `chore/upgrade-vite`      |
| `docs/`        | Chỉ sửa tài liệu              | `docs/update-setup-guide` |
| `explore/`     | Thử nghiệm, có thể xóa        | `explore/try-nextjs`      |

Chữ thường, gạch ngang, không dấu. Commit theo Conventional Commits.
Muốn xem lại một phase mà không sửa: `git checkout <tag>`. Muốn thử nghiệm từ một phase: `git checkout -b explore/from-init <tag>`.

### Checkpoint

Mở session mới (`/clear`), hỏi Claude 3 câu, mỗi câu nhắm một luật:

| Câu hỏi                                                                    | Kết quả đúng                                          |
| -------------------------------------------------------------------------- | ----------------------------------------------------- |
| `Tóm tắt quy ước dự án này.`                                               | Nêu đúng stack, pnpm, cấu trúc, luật cấm, quy ước git |
| `Tôi muốn làm feature danh sách sản phẩm, đặt tên branch giúp tôi.`        | Đề xuất dạng `feat/product-list`                      |
| `Tôi muốn dùng một hàm trong features/cart cho shared/ui, import thế nào?` | Từ chối/cảnh báo, gợi ý chuyển sang `shared/lib`      |

Câu nào sai thì sửa đúng dòng luật tương ứng trong CLAUDE.md rồi hỏi lại.

### Bài học

- Bản nháp `/init` đọc được config thật nhưng thường thừa (suy ra được từ code) và thiếu luật kiến trúc, quy ước của bạn.
- Nguyên tắc: "luôn đúng" vào CLAUDE.md, "bắt buộc" vào hook, "theo tình huống" vào skill.
- CLAUDE.md tốt là tích lũy từ lỗi thực tế: mỗi lần Claude sai lặp lại, thêm một dòng.
- Đừng đưa vào CLAUDE.md những nguyên tắc dành cho người làm (ví dụ "chưa build skill trước khi có mẫu"), vì Claude có thể từ chối đúng việc bạn nhờ.
- Đừng ghi trạng thái hiện tại ("phase 1 done") vào CLAUDE.md, nó lỗi thời rất nhanh. Thông tin này đã nằm ở `git log` và tag.

### Skill/Subagent ở phase này

Chưa có. Chưa có quy trình nào lặp lại đủ để đóng gói.

### Chốt phase

```bash
git add docs/SETUP-GUIDE.md
git commit -m "docs: document phase 2 in setup guide"
git checkout main
git merge --no-ff phase/02-structure-claude-md
git tag v0.2-structure-claude-md
```

Nếu làm qua PR trên GitHub: merge PR, `git checkout main && git pull`, rồi gắn tag trên `main`. Tag phải trỏ vào commit đã gồm cả `CLAUDE.md` và guide.

---

## Phase 3: Design tokens, Button mẫu, skill `new-component`

_Chưa làm._

## Phase 4: Testing, skill `write-tests`, agent `test-writer`

_Chưa làm._

## Phase 5: Hooks và permissions

_Chưa làm._

## Phase 6: Feature đầu tiên, agent `frontend-architect`, skill `new-feature`

_Chưa làm._

## Phase 7: Lớp review độc lập

_Chưa làm._

## Phase 8: MCP

_Chưa làm._

## Phase 9: CI và PR workflow

_Chưa làm._

## Phase 10: Chuẩn hóa cho team

_Chưa làm._

---

## Cách dùng tài liệu này với git

- Mỗi phase hoàn thành: cập nhật phần tương ứng trong file này, commit cùng code của phase đó, rồi gắn tag.
- Muốn xem source ở trạng thái của một phase: `git checkout <tag>` (ví dụ `git checkout v0.2-structure-claude-md`), rồi đọc phase tương ứng.
- Quay lại nhánh chính: `git checkout main`.
