# ĐẶC TẢ LUỒNG HOẠT ĐỘNG SWL 2.5: QUẢN LÝ TRẠNG THÁI LÔ THUỐC — KHÓA / MỞ KHÓA LÔ
## (MANAGE BATCH STATUS WORKFLOW — BF-05: BLOCK / UNBLOCK)

> **Dự án:** PharmaFlow — Hệ thống Quản lý Kho & Bán hàng Thông minh cho Nhà thuốc Tư nhân  
> **Mã luồng:** `SWL-2.5` | **Mã tiến trình Draw.io:** `2.5` (`BF-05 Manage Batch Status`)  
> **Phân hệ:** Quản lý Kho Dược & Kiểm soát Chất lượng Thuốc (Quality Control & Batch Management)  
> **Mã tài liệu:** `02_Requirement/7mainflow/Description SWL 5.md`  
> **Phiên bản:** v1.0 | **Ngày ban hành:** 13/10/2026  

---

## 1. TỔNG QUAN NGHIỆP VỤ (BUSINESS OVERVIEW)

### 1.1. Mục tiêu & Ý nghĩa
Luồng **SWL 2.5 (BF-05)** là chốt chặn quan trọng trong quy trình đảm bảo an toàn dược học và tuân thủ quy định thanh tra GPP của Bộ Y tế:
- Cho phép Dược sĩ có thẩm quyền hoặc Chủ nhà thuốc (Admin) chủ động can thiệp **Khóa tạm thời (BLOCK)** hoặc **Mở khóa (UNBLOCK)** một lô thuốc cụ thể trong kho.
- Các tình huống bắt buộc khóa lô:
  1. Có văn bản khẩn cấp thu hồi thuốc từ Cục Quản lý Dược (Bộ Y tế) do lô thuốc không đạt tiêu chuẩn chất lượng.
  2. Dược sĩ phát hiện lô thuốc tại quầy có hiện tượng biến màu, kết tủa, ẩm mốc, hoặc rách bao bì cần kiểm định lại.
  3. Lô thuốc có khiếu nại từ bệnh nhân đang trong quá trình điều tra nguyên nhân.
- Khi một lô bị **Khóa (BLOCKED)**, hệ thống lập tức loại bỏ lô đó khỏi thuật toán phân bổ FEFO tại quầy POS (**SWL 2.3**) và đơn hàng online (**SWL 2.2**), ngăn chặn hoàn toàn rủi ro bán nhầm thuốc lỗi cho bệnh nhân.

### 1.2. Phạm vi & Tác nhân tham gia
- **Dược sĩ có thẩm quyền / Chủ nhà thuốc (Authorized Staff / Owner-Admin):** Tra cứu kho theo sản phẩm/số lô, xem chi tiết hạn dùng, ra quyết định khóa hoặc mở khóa lô, nhập giải trình lý do chuyên môn.
- **Hệ thống PharmaFlow (System):** Xác thực quyền thao tác (RBAC), kiểm tra điều kiện chuyển đổi trạng thái (State Transition), cập nhật trạng thái lô thuốc (`ACTIVE` $\leftrightarrow$ `BLOCKED`), ghi nhận nhật ký kiểm toán và làm mới trạng thái tồn kho khả dụng.

### 1.3. Căn cứ pháp lý & Quy tắc nghiệp vụ (Business Rules)
- **BR18 (Cấm bán thuốc không đạt chuẩn):** Thuốc thuộc lô bị khóa (`status = 'BLOCKED'`) tuyệt đối không được hiển thị hoặc gợi ý trên màn hình POS và storefront online.
- **BR11 (Kiểm soát FEFO):** Thuật toán FEFO chỉ được quét các lô có `status = 'ACTIVE'`. Lô `BLOCKED` bị loại trừ hoàn toàn khỏi danh sách ứng viên xuất kho.
- **Audit Trail (Nhật ký kiểm toán GPP):** Mọi thao tác đổi trạng thái lô bắt buộc lưu vết: `User_ID`, `Batch_ID`, `Old_Status`, `New_Status`, `Reason_Note`, `Timestamp`.

---

## 2. SƠ ĐỒ TIẾN TRÌNH HOẠT ĐỘNG (WORKFLOW DIAGRAM)

*Sơ đồ ánh xạ trực tiếp từ trang `2.5` trong tệp thiết kế `pharma.drawio`:*

```mermaid
flowchart TD
    Start([Dược sĩ vào Quản lý Kho]) --> StepViewInv[Xem Danh sách Tồn kho - View Inventory]
    StepViewInv --> StepSelectBatch[Chọn Sản phẩm / Lô Thuốc Cần kiểm tra - Select Product / Batch]
    StepSelectBatch --> StepReviewDetail[Xem Chi tiết Lô: HSD, Tồn kho, Nguồn gốc - Review Batch Detail]
    
    StepReviewDetail --> DecChangeStatus{Có Yêu cầu Đổi Trạng thái Khóa? - Change Block Status?}
    
    DecChangeStatus -- "Không" --> StepViewUpdated[Xem Lại Tồn kho Hiện tại - View Updated Inventory] --> EndNoChange([Kết thúc - Giữ nguyên trạng thái])

    DecChangeStatus -- "Có" --> StepSelectAction[Chọn Thao tác: Khóa Lô (BLOCK) hoặc Mở Khóa (UNBLOCK)]
    StepSelectAction --> SysValidatePerm[Hệ thống Kiểm tra Quyền Hạn & Tính Hợp lệ Chuyển Trạng thái]
    
    SysValidatePerm --> DecPermAllowed{Cho phép Thực hiện? - Allowed?}
    
    DecPermAllowed -- "Không (Không đủ quyền / Lô đã Hết hạn)" --> SysShowError[Hiển thị Thông báo Lỗi - Show Error / No Change] --> EndNoChange

    DecPermAllowed -- "Cho phép" --> SysUpdateStatus[Cập nhật Trạng thái Lô: ACTIVE <-> BLOCKED]
    SysUpdateStatus --> SysRecordMovement["Ghi Nhật ký Biến động & Lịch sử: BLOCK / UNBLOCK Movement + Audit"]
    SysRecordMovement --> SysRefreshState[Làm mới Trạng thái Kho & Trừ Tồn Khả dụng Khỏi POS]
    SysRefreshState --> StepViewUpdated --> EndSuccess([Hoàn tất Cập nhật Trạng thái Lô])
```

---

## 3. BẢNG KỊCH BẢN THỰC THI TỪNG BƯỚC (STEP-BY-STEP SCENARIO TABLE)

| Bước | Tác nhân | Hành động & Thao tác Giao diện | Xử lý Hệ thống & Logic Backend | Ràng buộc (BR) | Kết quả đầu ra |
|:---:|:---|:---|:---|:---:|:---|
| **1** | Dược sĩ / Admin | Truy cập màn hình Quản lý kho (`admin/inventory-list.jsp`) | `InventoryServlet` (GET): Nạp danh sách các sản phẩm, tổng tồn kho và số lượng lô tương ứng | — | Bảng tổng quan tồn kho hiển thị đầy đủ |
| **2** | Dược sĩ / Admin | Nhấn vào một sản phẩm để xem danh sách các lô thuốc (`admin/inventory-batch.jsp`) | Truy vấn bảng `inventory_batches` theo `product_id`: lấy `batch_number`, `expiry_date`, `on_hand_quantity`, `status` | BR11 | Hiển thị chi tiết từng lô với badge trạng thái: `Đang bán` (Active) hoặc `Tạm khóa` (Blocked) |
| **3** | Dược sĩ / Admin | Nhấn vào nút chi tiết lô thuốc cần kiểm tra | Hiển thị modal lịch sử nhập kho, nhà cung cấp ban đầu, các lần xuất bán liên quan | Audit Trail | Dược sĩ nắm toàn bộ thông tin nguồn gốc lô thuốc |
| **4** | Dược sĩ / Admin | Đánh giá tình huống: Có văn bản thu hồi hoặc phát hiện thuốc hỏng $\rightarrow$ Nhấn **"Khóa lô thuốc"** (hoặc **"Mở khóa"**) | Mở form xác nhận: Yêu cầu chọn Lý do khóa (Thu hồi Cục Dược, Nghi ngờ chất lượng, Hỏng bao bì, Khác) và nhập ghi chú chi tiết | — | Form nhập giải trình xuất hiện |
| **5** | Hệ thống | Kiểm tra quyền hạn người dùng (RBAC) | Chỉ tài khoản có vai trò `ADMIN` hoặc `PHARMACIST` được phép; tài khoản `STAFF` thu ngân thông thường bị từ chối | RBAC | Chặn người dùng không có thẩm quyền |
| **6** | Hệ thống | Kiểm tra tính hợp lệ của việc chuyển trạng thái | - Nếu mở khóa (UNBLOCK): Lô đó phải có HSD còn lại $> 0$ ngày (không được mở khóa lô đã hết hạn).<br/>- Nếu khóa (BLOCK): Lô đang ở trạng thái `ACTIVE`. | BR18 | Đảm bảo chuyển đổi trạng thái hợp lệ |
| **7** | Dược sĩ / Admin | Nhấn nút **"Xác nhận Thay đổi Trạng thái"** | `InventoryServlet` (POST `action=change-batch-status`): Bắt đầu mở Transaction cơ sở dữ liệu | — | Bắt đầu xử lý ghi nhận dữ liệu |
| **8** | Hệ thống | Cập nhật trường `status` của lô thuốc trong `inventory_batches` | Chuyển `status = 'BLOCKED'` (hoặc `status = 'ACTIVE'`) | BR18 | Lô thuốc được cập nhật trạng thái mới |
| **9** | Hệ thống | Ghi bản ghi biến động và nhật ký kiểm toán (Audit History) | Insert bảng `inventory_movements` (`movement_type = 'BATCH_BLOCK'` hoặc `'BATCH_UNBLOCK'`) kèm `reason` và `user_id` | Audit Trail | Lưu vết vĩnh viễn phục vụ thanh tra GPP |
| **10** | Hệ thống | Làm mới trạng thái tồn kho toàn hệ thống | Tự động loại trừ số lượng của lô này khỏi `sellable_stock` trên POS và Storefront | BR11, BR18 | Quầy POS lập tức không còn thấy lô này để bán |
| **11** | Dược sĩ / Admin | Nhận thông báo thành công trên giao diện | Hiển thị badge đỏ `ĐÃ KHÓA (BLOCKED)` bên cạnh số lô thuốc | — | Hoàn tất thao tác nghiệp vụ an toàn |

---

## 4. ÁNH XẠ GIAO DIỆN & LUỒNG MÀN HÌNH (UI/UX SCREEN FLOW)

### 4.1. Màn hình liên quan
- **`SCR_ADM_INVENTORY_LIST` (`admin/inventory-list.jsp`):** Danh mục tồn kho theo thuốc.
- **`SCR_ADM_INVENTORY_BATCH` (`admin/inventory-batch.jsp`):** Danh sách các lô của một thuốc, có nút hành động [Khóa] / [Mở khóa] tương ứng từng lô.
- **`MODAL_BATCH_STATUS_CHANGE`:** Popup cảnh báo xác nhận đổi trạng thái lô, có trường chọn lý do (Dropdown) và ô nhập ghi chú giải trình.

### 4.2. Các phần tử tương tác quan trọng
- **Badge trạng thái lô:**
  - `ACTIVE`: Màu xanh lá cây — Đang lưu hành bình thường.
  - `BLOCKED`: Màu đỏ đậm — Tạm khóa bán, kèm biểu tượng ổ khóa.
  - `EXPIRED`: Màu xám — Đã hết hạn sử dụng.
- **Nút "Khóa lô":** Yêu cầu xác nhận 2 bước để tránh bấm nhầm làm gián đoạn bán hàng.

---

## 5. ĐẶC TẢ KỸ THUẬT & SOURCE CODE MAPPING

### 5.1. Controller & Backend Handling
- **Controller chính:** `src/java/controller/InventoryServlet.java`.
- **Luồng xử lý POST thay đổi trạng thái lô (`handleChangeStatus`):**
  ```java
  int batchId = Integer.parseInt(request.getParameter("batchId"));
  String newStatus = request.getParameter("status"); // "BLOCKED" hoặc "ACTIVE"
  String reason = request.getParameter("reason");
  int userId = ((User) session.getAttribute("user")).getId();

  Connection conn = DBContext.getConnection();
  try {
      conn.setAutoCommit(false);
      InventoryBatch batch = batchDAO.getBatchById(conn, batchId);
      
      // Không cho phép mở khóa nếu lô đã hết hạn sử dụng
      if ("ACTIVE".equals(newStatus) && batch.getExpiryDate().isBefore(LocalDate.now())) {
          throw new IllegalStateException("Không thể mở khóa lô thuốc đã quá hạn sử dụng");
      }
      
      batchDAO.updateStatus(conn, batchId, newStatus);
      movementDAO.recordMovement(conn, batchId, "STATUS_CHANGE_" + newStatus, 0, userId, reason);
      conn.commit();
  } catch (Exception e) {
      conn.rollback();
  }
  ```

### 5.2. Bảng cơ sở dữ liệu tác động
- `inventory_batches`: Cập nhật cột `status` (`ACTIVE`, `BLOCKED`, `EXPIRED`).
- `inventory_movements`: Ghi nhật ký biến động trạng thái kèm lý do giải trình.

---

## 6. LUỒNG NGOẠI LỆ & RỦI RO (ALTERNATIVE & EXCEPTION FLOWS)

| Mã ngoại lệ | Tình huống phát sinh | Cách xử lý hệ thống |
|:---:|:---|:---|
| **EX-2.5.1** | Dược sĩ cố tình mở khóa một lô thuốc đã quá hạn sử dụng | Hệ thống chặn thao tác, hiển thị thông báo lỗi nghiêm trọng: *"Lô thuốc đã hết hạn ngày [ExpiryDate], quy định GPP nghiêm cấm mở khóa để bán. Vui lòng lập phiếu xuất hủy hoặc trả NCC (SWL 2.7)"*. |
| **EX-2.5.2** | Lô thuốc đang được giữ chỗ trong một đơn hàng online chưa dispatch (SWL 2.1/2.2) | Hệ thống cảnh báo: *"Lô thuốc này đang có [Số lượng] đơn vị được giữ chỗ trong Đơn hàng #[OrderId]".* Cho phép khóa lô đồng thời tự động hủy phân bổ của lô đó để nhân viên nhặt hàng chọn lô khác thay thế. |

---

## 7. TIÊU CHÍ NGHIỆM THU (ACCEPTANCE CRITERIA)
- [x] Chỉ người dùng có vai trò phù hợp mới nhìn thấy và bấm được nút Khóa/Mở khóa lô.
- [x] Lô sau khi bị khóa lập tức không thể được tìm thấy hay tự động gán vào giỏ hàng POS hoặc đơn online.
- [x] Không thể mở khóa một lô thuốc đã hết hạn sử dụng.
- [x] Toàn bộ thao tác khóa và mở khóa đều lưu vết đầy đủ trong nhật ký biến động kho kèm lý do giải trình.
