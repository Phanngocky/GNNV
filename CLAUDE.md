# CLAUDE.md - Giảng Viên Second Brain Operating Manual

You are going to help me build and maintain a Second Brain. This is a persistent, compounding knowledge system. Instead of retrieving from raw documents at query time, you will incrementally build and maintain a structured knowledge base.

## 1. Cấu trúc 3 Lớp (Three Layers)
**Layer 1: 00 - Inbox** — Khu vực chứa dữ liệu thô. Immutable (không bao giờ sửa file gốc ở đây).
**Layer 2: 02 - Knowledge** — Lớp Tri thức do AI sở hữu và quản lý (tạo page, update link).
**Layer 3: CLAUDE.md** — Lớp Luật. Bạn phải đọc file này đầu tiên trong mỗi phiên làm việc.

## 2. Four Core Workflows (4 Quy trình Cốt lõi)

### 2.1 Ingest (Tiêu hóa dữ liệu)
Chạy quy trình này khi tôi ném một file vào `00 - Inbox/` và yêu cầu xử lý.
1. Đọc toàn bộ source file.
2. Thảo luận với tôi: Đâu là 2-3 điểm quan trọng nhất?
3. Viết `02 - Knowledge/sources/<Title>.md`.
4. Tạo/Cập nhật các concept liên quan trong `02 - Knowledge/concepts/`.
5. Cập nhật `02 - Knowledge/index.md`.
6. Ghi vết vào `02 - Knowledge/log.md`. **Bước 2 là bắt buộc. Không được bỏ qua.**

### 2.2 Query (Truy vấn)
Khi tôi hỏi, hãy đọc `index.md`, tìm page liên quan, trả lời có trích dẫn `[[Page]]`.

### 2.3 Lint (Dọn dẹp hệ thống)
Tìm các mâu thuẫn, file mồ côi (orphan pages), broken links và báo cáo để sửa chữa.

### 2.4 Review (Đánh giá)
Đánh giá định kỳ các project đang active, các file chưa xử lý trong Inbox.

## 3. Page Conventions & Syntax

**Frontmatter Bắt buộc:** (Trừ index.md và log.md)
---
type: source | concept | synthesis
title: "Tên hiển thị tiếng Việt có dấu"
sources: 1
tags: [tag1, tag2]
updated: YYYY-MM-DD
---

**Quy tắc Đặt tên File (Naming):**
- Source: Kebab-case (VD: `phuong-phap-nghien-cuu.md`)
- Concept: lowercase-hyphens (VD: `tu-duy-he-thong.md`)
- Luôn sử dụng Wiki-links: `[[PageName]]`.

## 4. Log Format (Cú pháp Ghi Log)
Nối thêm (append) vào `02 - Knowledge/log.md` theo ĐÚNG định dạng:
`## [YYYY-MM-DD] <type> | <title>`
(Types: setup, ingest, query, lint, review, edit)

## 5. Working Principles (Nguyên tắc làm việc)
- **Propose before acting:** Đề xuất trước khi hành động.
- **Verification Gate:** Tự kiểm chứng tính chính xác của dữ liệu (không ảo giác) trước khi xuất kết quả.
- Tiến hóa: Gặp lỗi phải đúc kết vào `05 - Operations/patterns.md`.
