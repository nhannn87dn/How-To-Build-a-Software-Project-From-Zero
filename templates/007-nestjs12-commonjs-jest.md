# ADR-007: NestJS 12 ở dạng CommonJS, test bằng Jest, chạy trên Node 24

- **Trạng thái:** Đã chấp nhận (10/10/2026)
- **Ngày:** 10/10/2026

## Bối cảnh
NestJS 12 (phát hành 27/08/2026) làm các gói lõi thành ESM. CLI mới hỏi chọn hệ module khi tạo project: **ESM kèm Vitest** (mặc định) hoặc **CommonJS kèm Jest**. Quy ước của project dùng Jest. Công cụ CLI cần Node từ 24.15 trở lên.

## Quyết định
- Tạo project **CommonJS + Jest**.
- Chạy Jest bằng `node --experimental-vm-modules node_modules/jest/bin/jest.js` (đã gắn vào các script `test*`), vì Jest cần cờ này để nạp các gói ESM của Nest.
- Dùng **Node 24** (`.nvmrc`, `engines >= 24.15.0`).
- Lint bằng **oxlint**, định dạng bằng Prettier (mặc định của CLI 12).
- `tsconfig.json` thêm `"rootDir": "."` để `ts-jest` chạy được với TypeScript 6 (không thêm vào `tsconfig.build.json` để `dist/main.js` giữ nguyên vị trí).

## Các phương án đã cân nhắc
| Phương án | Ưu | Nhược |
|---|---|---|
| **CommonJS + Jest** | Giữ quy ước Jest; import tương đối không cần đuôi `.js`; ít rủi ro với quan hệ entity hai chiều của TypeORM | Jest phải dùng cờ thử nghiệm và in cảnh báo mỗi lần chạy; cần Node 24 |
| ESM + Vitest | Mặc định của Nest 12, không cần cờ thử nghiệm | Đổi quy ước test sang Vitest; mọi import tương đối phải có đuôi `.js`; entity TypeORM có quan hệ vòng dễ gặp lỗi khởi tạo khi chạy ESM (cần bọc kiểu `Relation<>`) |

## Hệ quả
- Test chỉ chạy được trên Node 24 trở lên; CI phải dùng cùng phiên bản
- Khi Jest hỗ trợ nạp ESM ổn định không cần cờ, bỏ cờ trong các script
- Không viết thêm cấu hình `transformIgnorePatterns` cho gói ESM: đã thử và không chạy được với gói khai báo `"type": "module"`

## Xem lại khi
Gặp trở ngại thật sự với Jest và cờ ESM, hoặc khi chuyển cả dự án sang ESM (khi đó đổi sang Vitest cùng lúc).
