Hãy khởi tạo một dự án frontend từ đầu trong thư mục hiện tại (đang trống), theo đúng đặc tả dưới đây. Làm tuần tự, sau mỗi bước chạy lệnh kiểm tra tương ứng. Không cài thêm bất kỳ thư viện nào ngoài danh sách. Không tạo CLAUDE.md, .claude/, Tailwind, test framework hay Storybook (đó là các phase sau).

## Mục tiêu

Repo có "guardrails" tự động: TypeScript strict, ESLint, Prettier, và git hook chặn commit nếu code lỗi.

## Stack

Vite + React + TypeScript, package manager là pnpm (không dùng npm/yarn), ESLint, Prettier, Husky, lint-staged.

## Các bước

1. Khởi tạo
   - Tạo project Vite template `react-ts` ngay trong thư mục hiện tại (dùng chế độ không tương tác, không tạo thư mục con). Tên package: `react-claude-code`.
   - `git init` ngay trong thư mục có package.json, rồi `pnpm install`.

2. TypeScript strict
   - Trong `tsconfig.app.json`, đảm bảo `compilerOptions` có `"strict": true` và `"noUncheckedIndexedAccess": true`. Giữ nguyên các option sẵn có, chỉ thêm cái còn thiếu.

3. Cài dev dependencies
   - `pnpm add -D husky lint-staged prettier eslint-config-prettier`

4. Scripts trong package.json (giữ các script sẵn có, bổ sung nếu thiếu)
   - "typecheck": "tsc -b --noEmit"
   - "format": "prettier --write ."
   - "prepare": "husky"
   - đảm bảo có: dev, build, lint, preview

5. Prettier
   - `.prettierrc`: semi true, singleQuote true, trailingComma "all", printWidth 100.
   - `.prettierignore`: dist, node_modules, pnpm-lock.yaml.

6. Nối Prettier với ESLint
   - Trong `eslint.config.js`, import `eslint-config-prettier` và đặt nó là phần tử CUỐI CÙNG của mảng config (nếu file dùng tseslint.config(...) thì thêm vào cuối lời gọi đó). Giữ nguyên config hiện có.

7. lint-staged (trong package.json, cấp ngoài cùng, không nằm trong scripts)
   - "*.{ts,tsx}": ["eslint --fix", "prettier --write"]
   - "*.{json,md,css}": ["prettier --write"]

8. Husky
   - Chạy `pnpm exec husky init`.
   - Ghi đè `.husky/pre-commit` với đúng 2 dòng:
     pnpm lint-staged
     pnpm typecheck
   - Xác nhận `git config core.hooksPath` in ra `.husky/_`.

9. Chạy `pnpm format` một lần để toàn bộ file hiện có đúng chuẩn prettier.

## Kiểm tra (bắt buộc, báo kết quả từng mục)

a) `pnpm typecheck && pnpm lint && pnpm build` chạy sạch.
b) Kiểm tra strict: tạo tạm `src/test-ts.ts` gồm
const items = [1, 2, 3];
const first: number = items[0];
`pnpm typecheck` PHẢI báo lỗi (number | undefined). Xác nhận rồi xóa file này.
c) Kiểm tra hook: tạm thời thêm lỗi cú pháp vào một file .tsx, `git add .`, thử `git commit`. Commit PHẢI bị chặn. Sau đó hoàn tác lỗi.
d) Chạy lại (a) cho sạch, rồi commit: "chore: init project with guardrails". Không dùng --no-verify.

## Quy tắc

- Nếu một lệnh lỗi, đọc lỗi và sửa nguyên nhân gốc, không bỏ qua bước, không dùng --no-verify, không tắt rule.
- Không thêm tính năng hay file ngoài đặc tả.
- Khi xong, in ra: cây thư mục cấp 2, nội dung package.json (scripts + lint-staged), nội dung .husky/pre-commit, và kết quả 4 mục kiểm tra.
