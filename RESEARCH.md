# Research notes — vì sao fork GramJS thành lib-telegram-auth

**Ngày:** 2026-09-25
**Nguồn gốc:** fork từ [gram-js/gramjs](https://github.com/gram-js/gramjs) (upstream).
**Mục đích:** dùng cho use-case tương tự `share-fp2group-facebook` (đăng bài vào Telegram group/channel), nhưng KHÔNG cần browser automation (Cloak/Playwright) như Facebook.

## Bài toán gốc

Câu hỏi: Telegram có cần login mỗi session (giống Facebook qua Cloak) không, hay login 1 lần rồi share cho nhiều session khác?

## Kết luận research

Telegram có API chính thức (MTProto) — khác Facebook (không có official automation API cho việc này, phải giả lập browser + share cookie qua CloakBrowser).

- **Không cần browser giả lập.** MTProto client (GramJS/Telethon/Pyrogram) nói chuyện thẳng với Telegram server qua protocol chính thức, không phải scrape web UI → không lo Cloudflare/fingerprint detector.
- **Login 1 lần, share N nơi:** sau khi login (số điện thoại + OTP/2FA) 1 lần, gọi `client.session.save()` lấy ra `StringSession` — 1 chuỗi string đại diện toàn bộ auth. Chuỗi này pass vào `TelegramClient` khác (`new TelegramClient(new StringSession(savedSession), apiId, apiHash)`) là connect thẳng, không hỏi lại code.
- **Risk thấp hơn Facebook:** dùng account cá nhân qua API official được Telegram cho phép (userbot), miễn không spam/flood. Facebook cấm automation ngoài Graph API nên phải né bằng CloakBrowser.

## Rủi ro / lưu ý khi dùng

- `StringSession` = full quyền truy cập account, tương đương session cookie → tuyệt đối không log/print ra chat, không commit vào git. Lưu vault/`.env` local theo rule `vault-no-mcp` + `secrets-no-printout` của repo chính (bds-hue).
- Nhiều connection cùng 1 session string chạy đồng thời OK cho gửi tin/đăng group thông thường; tránh dùng cho voice/call layer (Telegram giới hạn concurrent ở layer đó).
- 2FA/OTP chỉ cần nhập lúc tạo session string lần đầu (interactive, 1 lần), sau đó tái dùng tới khi revoke.

## Sources

- [GramJS Authentication docs](https://painor.gitbook.io/gramjs/getting-started/authorization)
- [gram-js/gramjs GitHub](https://github.com/gram-js/gramjs)
- [MTProto vs HTTP Bot API — Telethon wiki](https://github.com/LonamiWebs/Telethon/wiki/MTProto-vs-HTTP-Bot-API)
- [Multiple connections · Issue #191 · gram-js/gramjs](https://github.com/gram-js/gramjs/issues/191)

## Cách sync lại từ upstream sau này

```bash
git remote add upstream https://github.com/gram-js/gramjs.git   # 1 lần
git fetch upstream
git merge upstream/master   # hoặc rebase, tùy nhu cầu
```

## Việc còn lại (chưa làm trong fork này)

- Chưa tích hợp vào `ts-platform` (bds-hue monorepo). Khi cần feature "share sang Telegram group", tạo service riêng `share-fp2group-telegram` trong `ts-platform/services/`, import lib này qua npm (sau khi publish riêng hoặc git dependency), KHÔNG vendor code trực tiếp vào monorepo.
- Chưa đổi tên package trong `package.json` (còn nguyên `telegram` từ upstream) — cân nhắc đổi khi thật sự publish riêng.
