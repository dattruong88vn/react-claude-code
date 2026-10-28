# Setup Guide: FE source với Claude Code

Tài liệu này ghi lại **từng bước dựng source từ đầu**, mỗi phase tương ứng một mốc (branch/tag) trong git. Checkout về mốc nào thì đọc phase đó để hiểu trạng thái source lúc ấy và lý do đằng sau.

| Phase | Nội dung                                                            | Tag gợi ý                  | Trạng thái |
| ----- | ------------------------------------------------------------------- | -------------------------- | ---------- |
| 1     | Init repo + guardrails                                              | `v0.1-init-guardrails`     | Hoàn thành |
| 2     | Cấu trúc thư mục + CLAUDE.md                                        | `v0.2-structure-claude-md` | Chưa làm   |
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

_Chưa làm. Sẽ bổ sung khi hoàn thành._

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
- Muốn xem source ở trạng thái của một phase: `git checkout <tag>` (ví dụ `git checkout v0.1-init-guardrails`), rồi đọc phase tương ứng.
- Quay lại nhánh chính: `git checkout main`.
