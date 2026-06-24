Bài 1: Tích Hợp & Sinh Code - Import Giao Dịch JSON

Phân Tích Các Trường Hợp Biên Khi Đọc JSON Từ Hệ Thống Cũ
Dữ liệu JSON từ hệ thống Core Banking cũ có thể chứa 6 nhóm lỗi biên chính:

Lỗi cấu trúc JSON: Toàn bộ chuỗi JSON bị sai cú pháp (thiếu dấu ngoặc đóng,

dấu phẩy thừa) - Jackson sẽ ném JsonParseException ngay khi bắt đầu parse,

cần bắt riêng để không làm sập toàn bộ tiến trình.
Trường bắt buộc bị null hoặc thiếu: id không tồn tại trong JSON object,

hoặc tồn tại nhưng có giá trị null hoặc chuỗi rỗng -

giao dịch không có ID không thể định danh được, phải bỏ qua.
Kiểu dữ liệu sai: amount là chuỗi "abc" thay vì số,

hoặc là boolean true - Jackson sẽ ném JsonMappingException khi map vào double,

cần bắt ở cấp từng phần tử thay vì để sập toàn batch.
Vi phạm điều kiện nghiệp vụ: amount là số hợp lệ về kiểu nhưng âm hoặc bằng 0,

status là giá trị không nằm trong tập hợp cho phép (PENDING, PROCESSING, v.v.) -

đây là lỗi logic nghiệp vụ, không phải lỗi kỹ thuật, cần xử lý riêng.
Định dạng ngày tháng không nhất quán: Hệ thống cũ có thể gửi transactionDate

theo nhiều format khác nhau như dd/MM/yyyy, yyyy-MM-dd, epoch timestamp -

cần thử parse theo thứ tự ưu tiên và bỏ qua nếu không parse được.
Null toàn bộ input: Chuỗi JSON đầu vào là null hoặc rỗng -

cần kiểm tra ngay đầu hàm trước khi truyền vào Jackson.


Prompt Tối Ưu
Hãy đóng vai một Senior Java Developer có kinh nghiệm tích hợp hệ thống tài chính,
chuyên xử lý dữ liệu từ hệ thống Core Banking legacy.

[INPUT - Đầu vào]
Một chuỗi JSON thô đại diện cho danh sách giao dịch từ hệ thống cũ.
Mỗi phần tử giao dịch gồm 4 trường: id (String), amount (số tiền),
status (trạng thái giao dịch), transactionDate (ngày giao dịch dạng ISO-8601).

Dữ liệu đầu vào CỐ TÌNH chứa các lỗi biên sau để kiểm thử:
- Một phần tử thiếu trường id hoặc id là chuỗi rỗng
- Một phần tử có amount là chuỗi chữ "abc" (sai kiểu dữ liệu)
- Một phần tử có amount âm (-500)
- Một phần tử có status là "PENDING" (không nằm trong tập hợp hợp lệ)
- Một phần tử có transactionDate sai định dạng ("32/13/2024")
- Một phần tử hợp lệ hoàn toàn để xác nhận hệ thống vẫn hoạt động đúng

[OUTPUT - Đầu ra]
Trả về List<TransactionDTO> chỉ chứa các giao dịch hợp lệ đã qua kiểm duyệt.
Định nghĩa TransactionDTO là Java record với các trường:
- String id
- BigDecimal amount (dùng BigDecimal thay double để tránh sai số tiền tệ)
- String status
- LocalDateTime transactionDate

[PROCESSING - Xử lý & Ràng buộc kỹ thuật]
- Sử dụng Jackson ObjectMapper của Spring Boot để parse chuỗi JSON
- Kiểm duyệt từng phần tử độc lập: lỗi một phần tử KHÔNG được làm sập toàn batch
- Điều kiện hợp lệ: amount > 0, status chỉ nhận "SUCCESS" hoặc "FAILED",
  id không được null hoặc rỗng, transactionDate phải parse được sang LocalDateTime
- Bỏ qua phần tử lỗi và ghi log.warn() kèm nội dung phần tử đó và lý do bỏ qua
- Bắt riêng JsonParseException khi toàn bộ chuỗi JSON bị lỗi cú pháp
- Kiểm tra đầu vào null hoặc rỗng ngay đầu hàm

[LANGUAGE & TECH STACK]
- Java 17, Spring Boot 3.x
- Jackson ObjectMapper (com.fasterxml.jackson)
- Lombok @Slf4j để logging
- BigDecimal với RoundingMode.HALF_UP cho phép tính tiền
- Chú thích từng bước logic bằng tiếng Việt

[DRY-RUN CoT - Suy luận từng bước]
Đừng viết code ngay. Hãy thực hiện lần lượt:

Bước 1: Phác thảo thuật toán bằng pseudocode mô tả luồng xử lý từ đầu vào đến đầu ra.

Bước 2: Dry-run bằng văn bản với chuỗi JSON mẫu chứa đủ 6 loại lỗi biên đã mô tả ở Input.
Trình bày từng phần tử được xử lý thế nào, phần tử nào bị bỏ qua và lý do.

Bước 3: Chỉ sau khi hoàn thành 2 bước trên, sinh mã nguồn Java hoàn chỉnh.
Trình bày trong một khối code markdown duy nhất.

Phản Hồi Từ AI
Bước 1 - Pseudocode thuật toán:
PSEUDOCODE: TransactionParser.parse(String jsonString)
======================================================

FUNCTION parseTransactions(jsonString):

// Kiểm tra đầu vào
IF jsonString IS null OR jsonString IS blank:
log.warn("Chuỗi JSON đầu vào rỗng hoặc null")
RETURN empty list

// Parse toàn bộ chuỗi JSON thành mảng JsonNode
TRY:
rootArray = objectMapper.readTree(jsonString)
IF rootArray is NOT array:
THROW InvalidFormatException("Dữ liệu không phải mảng JSON")
CATCH JsonParseException:
log.error("Chuỗi JSON bị lỗi cú pháp toàn bộ")
THROW TransactionParseException wrapping original error

// Duyệt từng phần tử
result = new ArrayList()

FOR EACH node IN rootArray:
TRY:
// Bước 1: Kiểm tra id
id = node.get("id").asText()
IF id IS null OR id IS blank:
log.warn("Bỏ qua: id rỗng - node: {}", node)
CONTINUE

      // Bước 2: Kiểm tra amount kiểu dữ liệu và giá trị
      IF node.get("amount") IS NOT numeric:
        log.warn("Bỏ qua id={}: amount sai kiểu - {}", id, node)
        CONTINUE
      amount = parse BigDecimal from node.get("amount")
      IF amount <= 0:
        log.warn("Bỏ qua id={}: amount không hợp lệ - {}", id, amount)
        CONTINUE

      // Bước 3: Kiểm tra status
      status = node.get("status").asText()
      IF status NOT IN ["SUCCESS", "FAILED"]:
        log.warn("Bỏ qua id={}: status không hợp lệ - {}", id, status)
        CONTINUE

      // Bước 4: Parse transactionDate
      TRY:
        transactionDate = LocalDateTime.parse(node.get("transactionDate").asText())
      CATCH DateTimeParseException:
        log.warn("Bỏ qua id={}: transactionDate sai định dạng", id)
        CONTINUE

      // Tất cả hợp lệ - tạo DTO và thêm vào kết quả
      result.add(new TransactionDTO(id, amount, status, transactionDate))

    CATCH Exception e:
      log.warn("Bỏ qua phần tử lỗi không xác định: {}", e.getMessage())
      CONTINUE

log.info("Hoàn thành parse: {} hợp lệ / {} tổng", result.size(), rootArray.size())
RETURN result

Bước 2 - Dry-run với JSON mẫu:
JSON MẪU ĐẦU VÀO:
==================
[
{ "id": "TXN-001", "amount": 500000, "status": "SUCCESS",
"transactionDate": "2024-01-15T10:30:00" },
{ "id": "", "amount": 200000, "status": "SUCCESS",
"transactionDate": "2024-01-15T11:00:00" },
{ "id": "TXN-003", "amount": "abc", "status": "SUCCESS",
"transactionDate": "2024-01-15T11:30:00" },
{ "id": "TXN-004", "amount": -500, "status": "SUCCESS",
"transactionDate": "2024-01-15T12:00:00" },
{ "id": "TXN-005", "amount": 300000, "status": "PENDING",
"transactionDate": "2024-01-15T12:30:00" },
{ "id": "TXN-006", "amount": 150000, "status": "FAILED",
"transactionDate": "32/13/2024" }
]

XỬ LÝ TỪNG PHẦN TỬ:
====================

Phần tử 1 - TXN-001:
id      = "TXN-001"     → HỢP LỆ
amount  = 500000        → HỢP LỆ (số, > 0)
status  = "SUCCESS"     → HỢP LỆ
date    = "2024-01-15T10:30:00" → Parse LocalDateTime OK
KẾT QUẢ: THÊM VÀO DANH SÁCH ✓

Phần tử 2 - id rỗng:
id = ""  → VI PHẠM: id không được rỗng
log.warn("Bỏ qua: id rỗng - node: {...}")
KẾT QUẢ: BỎ QUA ✗

Phần tử 3 - TXN-003:
id     = "TXN-003"  → HỢP LỆ
amount = "abc"      → VI PHẠM: node không phải kiểu số (isNumber() = false)
log.warn("Bỏ qua id=TXN-003: amount sai kiểu - abc")
KẾT QUẢ: BỎ QUA ✗

Phần tử 4 - TXN-004:
id     = "TXN-004"  → HỢP LỆ
amount = -500       → VI PHẠM: amount <= 0
log.warn("Bỏ qua id=TXN-004: amount không hợp lệ - -500")
KẾT QUẢ: BỎ QUA ✗

Phần tử 5 - TXN-005:
id     = "TXN-005"  → HỢP LỆ
amount = 300000     → HỢP LỆ
status = "PENDING"  → VI PHẠM: không nằm trong [SUCCESS, FAILED]
log.warn("Bỏ qua id=TXN-005: status không hợp lệ - PENDING")
KẾT QUẢ: BỎ QUA ✗

Phần tử 6 - TXN-006:
id     = "TXN-006"  → HỢP LỆ
amount = 150000     → HỢP LỆ
status = "FAILED"   → HỢP LỆ
date   = "32/13/2024" → VI PHẠM: DateTimeParseException (ngày 32 tháng 13 không tồn tại)
log.warn("Bỏ qua id=TXN-006: transactionDate sai định dạng")
KẾT QUẢ: BỎ QUA ✗

TỔNG KẾT DRY-RUN:
Tổng phần tử: 6
Hợp lệ: 1 (TXN-001)
Bỏ qua: 5
log.info("Hoàn thành parse: 1 hợp lệ / 6 tổng")

Bước 3 - Mã nguồn Java hoàn chỉnh:
java// ===================== TRANSACTION DTO =====================

import java.math.BigDecimal;
import java.time.LocalDateTime;

/**
* Java record đại diện cho một giao dịch đã qua kiểm duyệt và hợp lệ.
* Dùng record để đảm bảo immutability - dữ liệu tài chính không nên bị thay đổi sau khi tạo.
* Dùng BigDecimal thay double để tránh sai số làm tròn trong phép tính tiền tệ.
  */
  public record TransactionDTO(
  String id,
  BigDecimal amount,
  String status,
  LocalDateTime transactionDate
  ) {}


// ===================== CUSTOM EXCEPTION =====================

/**
* Exception ném ra khi toàn bộ chuỗi JSON đầu vào bị lỗi cú pháp nghiêm trọng,
* không thể tiếp tục parse bất kỳ phần tử nào.
  */
  public class TransactionParseException extends RuntimeException {
  public TransactionParseException(String message, Throwable cause) {
  super(message, cause);
  }
  }


// ===================== TRANSACTION PARSER =====================

import com.fasterxml.jackson.core.JsonParseException;
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;

import java.math.BigDecimal;
import java.math.RoundingMode;
import java.time.LocalDateTime;
import java.time.format.DateTimeParseException;
import java.util.ArrayList;
import java.util.List;
import java.util.Set;

/**
* Parser chuyển đổi chuỗi JSON thô từ hệ thống Core Banking cũ
* thành danh sách TransactionDTO đã được kiểm duyệt.
*
* Nguyên tắc xử lý lỗi: lỗi một phần tử KHÔNG làm sập toàn batch,
* phần tử lỗi được bỏ qua kèm log cảnh báo, tiếp tục xử lý phần tử tiếp theo.
  */
  @Slf4j
  @Component
  public class TransactionParser {

  // Tập hợp các giá trị status hợp lệ theo quy định nghiệp vụ
  private static final Set<String> VALID_STATUSES = Set.of("SUCCESS", "FAILED");

  // ObjectMapper được tái sử dụng - thread-safe sau khi cấu hình xong
  private final ObjectMapper objectMapper;

  public TransactionParser() {
  this.objectMapper = new ObjectMapper();
  // Đăng ký module hỗ trợ LocalDateTime/LocalDate của Java 8+
  this.objectMapper.registerModule(new JavaTimeModule());
  }

  /**
    * Parse chuỗi JSON thô thành danh sách TransactionDTO hợp lệ.
    * Mỗi phần tử được kiểm duyệt độc lập - lỗi một phần tử không ảnh hưởng phần tử khác.
    *
    * @param jsonString Chuỗi JSON thô từ hệ thống Core Banking
    * @return Danh sách TransactionDTO đã qua kiểm duyệt, chỉ chứa phần tử hợp lệ
    * @throws TransactionParseException nếu toàn bộ chuỗi JSON bị lỗi cú pháp
      */
      public List<TransactionDTO> parseTransactions(String jsonString) {

      // Bước 1: Kiểm tra đầu vào không được null hoặc rỗng
      if (jsonString == null || jsonString.isBlank()) {
      log.warn("Chuỗi JSON đầu vào rỗng hoặc null - trả về danh sách rỗng.");
      return new ArrayList<>();
      }

      // Bước 2: Parse cấu trúc JSON tổng thể
      // Nếu cú pháp JSON sai hoàn toàn, không thể tiếp tục - ném exception
      JsonNode rootNode;
      try {
      rootNode = objectMapper.readTree(jsonString);
      } catch (JsonParseException e) {
      log.error("Chuỗi JSON bị lỗi cú pháp nghiêm trọng, không thể parse: {}",
      e.getMessage());
      throw new TransactionParseException(
      "Chuỗi JSON đầu vào bị lỗi cú pháp: " + e.getMessage(), e);
      } catch (Exception e) {
      log.error("Lỗi không xác định khi đọc chuỗi JSON: {}", e.getMessage(), e);
      throw new TransactionParseException("Lỗi đọc JSON: " + e.getMessage(), e);
      }

      // Bước 3: Kiểm tra root phải là mảng JSON
      if (!rootNode.isArray()) {
      log.error("Dữ liệu JSON không phải mảng - kiểu thực tế: {}", rootNode.getNodeType());
      throw new TransactionParseException(
      "Dữ liệu JSON phải là mảng, nhận được: " + rootNode.getNodeType(), null);
      }

      // Bước 4: Duyệt từng phần tử và kiểm duyệt độc lập
      List<TransactionDTO> result = new ArrayList<>();
      int totalCount = rootNode.size();

      log.info("Bắt đầu parse {} phần tử giao dịch từ JSON.", totalCount);

      for (int i = 0; i < totalCount; i++) {
      JsonNode node = rootNode.get(i);

           try {
               TransactionDTO dto = validateAndMap(node, i);
               if (dto != null) {
                   result.add(dto);
               }
           } catch (Exception e) {
               // Bắt mọi exception không lường trước để đảm bảo batch không bị sập
               log.warn("Bỏ qua phần tử index={} do lỗi không xác định: {} - node: {}",
                   i, e.getMessage(), node);
           }
      }

      log.info("Hoàn thành parse: {} hợp lệ / {} tổng phần tử.", result.size(), totalCount);

      return result;
      }

  /**
    * Kiểm duyệt và ánh xạ một JsonNode thành TransactionDTO.
    * Trả về null nếu phần tử vi phạm bất kỳ điều kiện nào.
    *
    * @param node  JsonNode đại diện cho một phần tử giao dịch
    * @param index Vị trí trong mảng để ghi log rõ ràng hơn
    * @return TransactionDTO nếu hợp lệ, null nếu cần bỏ qua
      */
      private TransactionDTO validateAndMap(JsonNode node, int index) {

      // Kiểm tra 1: Trường id không được null hoặc rỗng
      JsonNode idNode = node.get("id");
      if (idNode == null || idNode.isNull() || idNode.asText().isBlank()) {
      log.warn("Bỏ qua phần tử index={}: trường 'id' bị null hoặc rỗng - node: {}",
      index, node);
      return null;
      }
      String id = idNode.asText().trim();

      // Kiểm tra 2: Trường amount phải là kiểu số
      JsonNode amountNode = node.get("amount");
      if (amountNode == null || amountNode.isNull() || !amountNode.isNumber()) {
      log.warn("Bỏ qua id='{}': 'amount' không phải kiểu số - giá trị: {}",
      id, amountNode);
      return null;
      }

      // Kiểm tra 3: Giá trị amount phải lớn hơn 0
      BigDecimal amount = amountNode.decimalValue()
      .setScale(2, RoundingMode.HALF_UP);
      if (amount.compareTo(BigDecimal.ZERO) <= 0) {
      log.warn("Bỏ qua id='{}': 'amount' phải lớn hơn 0, nhận được: {}", id, amount);
      return null;
      }

      // Kiểm tra 4: Trường status phải nằm trong tập hợp hợp lệ
      JsonNode statusNode = node.get("status");
      if (statusNode == null || statusNode.isNull()) {
      log.warn("Bỏ qua id='{}': trường 'status' bị null.", id);
      return null;
      }
      String status = statusNode.asText().trim();
      if (!VALID_STATUSES.contains(status)) {
      log.warn("Bỏ qua id='{}': 'status' không hợp lệ - nhận được: '{}', hợp lệ: {}",
      id, status, VALID_STATUSES);
      return null;
      }

      // Kiểm tra 5: Trường transactionDate phải parse được sang LocalDateTime
      JsonNode dateNode = node.get("transactionDate");
      if (dateNode == null || dateNode.isNull() || dateNode.asText().isBlank()) {
      log.warn("Bỏ qua id='{}': trường 'transactionDate' bị null hoặc rỗng.", id);
      return null;
      }

      LocalDateTime transactionDate;
      try {
      transactionDate = LocalDateTime.parse(dateNode.asText().trim());
      } catch (DateTimeParseException e) {
      log.warn("Bỏ qua id='{}': 'transactionDate' sai định dạng ISO-8601 - nhận được: '{}'",
      id, dateNode.asText());
      return null;
      }

      // Tất cả kiểm duyệt đều qua - tạo TransactionDTO hợp lệ
      log.info("Giao dịch id='{}' hợp lệ - amount: {}, status: {}, date: {}",
      id, amount, status, transactionDate);

      return new TransactionDTO(id, amount, status, transactionDate);
      }
      }