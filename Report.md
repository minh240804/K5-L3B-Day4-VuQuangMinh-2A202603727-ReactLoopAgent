# Báo cáo cải thiện agent — case A1

## 1. Case được chọn

Case A1 yêu cầu đối chiếu GitHub Actions và Cloudflare Spectrum: xác định incident liên quan của từng dịch vụ, trạng thái, phạm vi ảnh hưởng và URL nguồn; đồng thời không suy luận hai incident có cùng nguyên nhân.

Snapshot được đánh giá tại `2026-11-09 03:50 UTC` (`10:50:47 +07:00` trong trace). Kỳ vọng là GitHub Actions có incident lịch sử `resolved`, Cloudflare có incident Spectrum tại Phoenix ở trạng thái `monitoring`, có URL trực tiếp cho cả hai và giữ nguyên ranh giới bằng chứng.

## 2. Kết quả trước khi cải thiện

### Evidence trong trace

Ở bước 1, agent gọi:

```text
get_status(provider="github", component="GitHub Actions")
get_status(provider="cloudflare", component="Cloudflare Spectrum")
```

GitHub trả về các incident Actions với `status: resolved`, sau đó bước 2 gọi `get_incident_details` cho 5 incident và lấy được URL, ví dụ `https://stspg.io/lv4q3xm9r0rk`.

Cloudflare trả về `incidents: []`, `matching_incidents_in_last_50: 0` và ghi rõ: “không tìm thấy không chứng minh không có sự cố”. Tuy nhiên agent vẫn kết luận “không có incident Spectrum”, đồng thời dẫn `https://www.cloudflarestatus.com` là URL tổng quan thay vì URL incident cụ thể.

### Lỗi agent

Đây là lỗi recall kết hợp với thiếu fallback. Bộ lọc tìm theo component không thấy bản ghi `Spectrum Availability Issue in Phoenix`; agent không gọi lại `get_status` ở cấp provider để lấy các incident đang mở và không gọi `get_incident_details` cho incident Cloudflare.

Điểm trước cải thiện: **66/100** — factuality `25/40`, completeness `14/25`, evidence `15/20`, boundaries `12/15`. Judge xác nhận phần GitHub đúng nhưng Cloudflare bị bỏ sót incident Phoenix và URL trực tiếp.

## 3. Thay đổi đã thực hiện

Chỉ sửa `SYSTEM_PROMPT` trong [agent.py](agent.py), không sửa schema, implementation hay dữ liệu của tool.

Các rule mới:

1. Kết quả rỗng trong tập giới hạn (ví dụ 50 incident gần nhất) chỉ có nghĩa là “chưa tìm thấy trong dữ liệu đã trả về”, không phải “không có incident” hoặc “chưa từng xảy ra”.
2. Nếu tìm theo tên component trả về 0 kết quả, **bắt buộc** gọi lại tool để lấy danh sách tất cả incident đang mở của provider, hoặc tra cứu trực tiếp status page chính thức bằng web.
3. Nếu không thực hiện được fallback, phải nói rõ là chưa đủ bằng chứng.
4. Chỉ xác nhận incident, trạng thái và phạm vi khi xuất hiện trực tiếp trong tool result.
5. Không suy đoán/gán quan hệ nhân quả giữa incident, provider hoặc dịch vụ.
6. Ưu tiên URL incident trực tiếp, không thay bằng URL trang tổng quan.

Đây là thay đổi ở tầng prompt, giữ nguyên tool và hạn chế can thiệp vào implementation.

## 4. Demo trace nhiều bước sau cải thiện

### Bước 1 — truy vấn theo component

```text
get_status(provider="github", component="GitHub Actions")
→ có các incident GitHub, matching_incidents_in_last_50 = 11

get_status(provider="cloudflare", component="Cloudflare Spectrum")
→ incidents = [], matching_incidents_in_last_50 = 0
```

Theo rule mới, kết quả Cloudflare chỉ là “chưa tìm thấy”, chưa được kết luận.

### Bước 2 — fallback cấp provider

```text
get_status(provider="cloudflare")
→ id: c60k3g4vsoc1
→ name: Spectrum Availability Issue in Phoenix
→ status: monitoring
```

### Bước 3 — lấy chi tiết

```text
get_incident_details(provider="cloudflare", incident_id="c60k3g4vsoc1")
→ components: ["Spectrum"]
→ status: monitoring
→ phạm vi: ảnh hưởng availability của ứng dụng Spectrum tại Phoenix
→ url: https://www.cloudflarestatus.com/incidents/c60k3g4vsoc1
```

Agent đồng thời lấy chi tiết GitHub, ví dụ incident `t89g2q1800fr` ở trạng thái `resolved`, URL `https://stspg.io/lv4q3xm9r0rk`.

### Bước 4 — tổng hợp

Agent trình bày độc lập: GitHub là incident đã resolved; Cloudflare Phoenix đang monitoring; không kết luận hai bên có cùng nguyên nhân.

## 5. Kết quả so sánh

Sau cải thiện, trace có fallback Cloudflare ở bước 2 và lấy chi tiết incident ở bước 3. Câu trả lời nhận diện đúng cả hai provider, phạm vi và URL trực tiếp:

- GitHub Actions: các incident liên quan ở trạng thái `resolved`.
- Cloudflare Spectrum: `Spectrum Availability Issue in Phoenix`, `monitoring`, component `Spectrum`, URL `https://www.cloudflarestatus.com/incidents/c60k3g4vsoc1`.
- Không có suy diễn nhân quả.

Điểm sau cải thiện: **100/100**; factuality `40/40`, completeness `25/25`, evidence `20/20`, boundaries `15/15`, mechanical violations `0`. Trace có `9` tool calls trong `3` bước tool. So với trước, điểm tăng **34 điểm (66 → 100)**.

Judge xác nhận kết quả sau đã đúng hai provider, đúng Phoenix đang monitoring, có URL cho cả hai và nêu đúng giới hạn tìm kiếm 50 incident.

## 6. Giới hạn

- Kết quả phụ thuộc snapshot, không đại diện cho trạng thái hiện tại ngoài thời điểm trên.
- Fallback cấp provider vẫn phụ thuộc phạm vi dữ liệu tool; kết quả rỗng sau fallback cũng không chứng minh incident chưa từng tồn tại.
- Prompt không bảo đảm fallback nếu provider lỗi, hết `max_steps`, hết quota hoặc tool không hỗ trợ truy vấn cấp provider/web.
- Web có thể gặp timeout, rate limit, thay đổi nội dung hoặc thiếu URL incident chuẩn hóa.
- GitHub có nhiều incident; liệt kê nhiều bản ghi giúp đầy đủ nhưng có thể rút gọn nếu chỉ cần một incident đại diện.
- Judge là LLM, nên điểm phải được đọc cùng trace và evidence; 100/100 không chứng minh đúng trong mọi case khác.

## 7. Kết luận

Lỗi ban đầu là agent dừng sau truy vấn component bị giới hạn và biến “không tìm thấy” thành “không có”. Rule fallback bắt buộc giúp agent lấy được incident Phoenix, URL trực tiếp và phạm vi ảnh hưởng mà không cần sửa tool. Với A1, kết quả tăng từ 66/100 lên 100/100; ranh giới quan trọng là vẫn phải nói rõ giới hạn dữ liệu và không suy đoán quan hệ nhân quả.
