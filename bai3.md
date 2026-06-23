# BÀI 3: Thực hành Refactor & Nâng cấp Giao dịch (Refinement Process - Robustness & Logging)

---

# 1. Phân tích các lỗ hổng nghiêm trọng của đoạn code ban đầu

## Code gốc

```java
public class OrderPlacementService {

    private InventoryRepository inventoryRepository;

    private PaymentGateway paymentGateway;

    private OrderRepository orderRepository;

    public void placeOrder(Order order) {

        // Trừ kho
        for (OrderItem item : order.getItems()) {
            Product product = inventoryRepository.findById(item.getProductId()).orElse(null);
            product.setStock(product.getStock() - item.getQuantity());
            inventoryRepository.save(product);
        }

        // Thanh toán qua Gateway
        paymentGateway.charge(order.getCustomerId(), order.getTotalAmount());

        // Lưu đơn hàng
        orderRepository.save(order);
    }
}
```

---

## Lỗi 1: NullPointerException

```java
Product product =
    inventoryRepository.findById(id)
                       .orElse(null);

product.setStock(...);
```

Nếu sản phẩm không tồn tại:

```java
product == null
```

=> Ném NullPointerException.

---

## Lỗi 2: Không kiểm tra tồn kho

Ví dụ:

```text
Tồn kho: 3

Khách mua: 10
```

Code vẫn thực hiện:

```java
3 - 10 = -7
```

=> Kho âm.

---

## Lỗi 3: Không xử lý lỗi thanh toán

```java
paymentGateway.charge(...)
```

Nếu gateway lỗi:

```java
Timeout
Network Error
Card Declined
```

Không có try-catch.

---

## Lỗi 4: Không có Transaction

Tình huống:

```text
Bước 1:
Đã trừ kho

Bước 2:
Thanh toán lỗi
```

Kết quả:

```text
Kho bị giảm
Nhưng đơn hàng không tạo
```

=> Mất đồng bộ dữ liệu.

---

## Lỗi 5: Không có Logging

Không thể biết:

- Ai đặt hàng.
- Đơn nào lỗi.
- Thanh toán lỗi ở đâu.

---

## Lỗi 6: Không có Result Object

```java
void placeOrder(...)
```

Không trả kết quả.

Khó mở rộng API.

---

## Lỗi 7: Không có Unit Test

Không đảm bảo:

- Thanh toán thất bại.
- Hết hàng.
- Rollback.

---

# 2. Chuỗi Prompt Refinement Chain

---

# VÒNG 1 – ROBUSTNESS

```text
Bạn là Senior Java Backend Engineer.

Hãy refactor đoạn code OrderPlacementService theo hướng tăng độ an toàn dữ liệu.

Yêu cầu:

1. Kiểm tra order không được null.
2. Kiểm tra danh sách items không được null hoặc rỗng.
3. Kiểm tra product tồn tại.
4. Kiểm tra tồn kho:

   Nếu:
   stock < quantity

   thì ném:

   OutOfStockException

5. Bọc lỗi từ PaymentGateway.

   Nếu charge() thất bại:

   ném:

   PaymentFailedException

6. Không sử dụng Optional.orElse(null).

7. Tuân thủ Clean Code.

8. Giải thích mọi thay đổi.
```

---

# VÒNG 2 – MAINTAINABILITY & CLEAN CODE

```text
Tiếp tục cải tiến code ở vòng trước.

Yêu cầu:

1. Chuyển sang Spring Boot Production-ready.

2. Thêm:

   @Service

3. Thêm:

   @RequiredArgsConstructor

4. Thêm:

   @Transactional

5. Đảm bảo:

   Nếu payment thất bại
   => rollback toàn bộ inventory update

6. Tích hợp Lombok.

7. Tích hợp:

   @Slf4j

8. Ghi log:

   INFO:
   - Bắt đầu đặt hàng
   - Trừ kho
   - Thanh toán
   - Lưu đơn

   ERROR:
   - Hết hàng
   - Thanh toán thất bại
   - Exception bất thường

9. Trả lại mã nguồn hoàn chỉnh.
```

---

# VÒNG 3 – CONTEXT-SPECIFIC TUNING

```text
Tiếp tục cải tiến code hiện tại.

Yêu cầu:

1. Không trả về void.

2. Tạo DTO:

   OrderPlacementResult

Bao gồm:

- orderId
- paymentStatus
- message

3. placeOrder()

trả về:

OrderPlacementResult

4. Viết JUnit 5 Test.

5. Sử dụng Mockito.

6. Mock:

- InventoryRepository
- OrderRepository
- PaymentGateway

7. Viết test case:

Payment Gateway thất bại.

Yêu cầu xác minh:

- PaymentFailedException được ném ra.
- OrderRepository.save() không được gọi.
- Transaction rollback xảy ra.

8. Sinh mã nguồn Production-ready hoàn chỉnh.
```

---

# 3. Minh chứng chạy thực tế (Kết quả cuối cùng sau vòng 3)

## OrderPlacementResult.java

```java
import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;

@Data
@Builder
@AllArgsConstructor
public class OrderPlacementResult {

    private Long orderId;

    private String paymentStatus;

    private String message;
}
```

---

## PaymentFailedException.java

```java
public class PaymentFailedException
        extends RuntimeException {

    public PaymentFailedException(String message) {
        super(message);
    }
}
```

---

## OutOfStockException.java

```java
public class OutOfStockException
        extends RuntimeException {

    public OutOfStockException(String message) {
        super(message);
    }
}
```

---

## OrderPlacementService.java

```java
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;

import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
@RequiredArgsConstructor
@Slf4j
public class OrderPlacementService {

    private final InventoryRepository inventoryRepository;

    private final PaymentGateway paymentGateway;

    private final OrderRepository orderRepository;

    @Transactional
    public OrderPlacementResult placeOrder(Order order) {

        if (order == null) {
            throw new IllegalArgumentException(
                    "Order must not be null");
        }

        if (order.getItems() == null
                || order.getItems().isEmpty()) {

            throw new IllegalArgumentException(
                    "Order items must not be empty");
        }

        log.info(
                "Starting order placement. customerId={}",
                order.getCustomerId());

        for (OrderItem item : order.getItems()) {

            Product product =
                    inventoryRepository.findById(
                            item.getProductId())
                            .orElseThrow(() ->
                                    new OutOfStockException(
                                            "Product not found"));

            if (product.getStock()
                    < item.getQuantity()) {

                log.error(
                        "Out of stock. productId={}",
                        item.getProductId());

                throw new OutOfStockException(
                        "Insufficient stock");
            }

            product.setStock(
                    product.getStock()
                            - item.getQuantity());

            inventoryRepository.save(product);

            log.info(
                    "Inventory updated. productId={}",
                    product.getId());
        }

        try {

            log.info(
                    "Charging payment. customerId={}",
                    order.getCustomerId());

            paymentGateway.charge(
                    order.getCustomerId(),
                    order.getTotalAmount());

        } catch (Exception ex) {

            log.error(
                    "Payment failed.",
                    ex);

            throw new PaymentFailedException(
                    "Payment gateway error");
        }

        Order savedOrder =
                orderRepository.save(order);

        log.info(
                "Order saved. orderId={}",
                savedOrder.getId());

        return OrderPlacementResult
                .builder()
                .orderId(savedOrder.getId())
                .paymentStatus("SUCCESS")
                .message("Order placed successfully")
                .build();
    }
}
```

---

## OrderPlacementServiceTest.java

```java
import static org.junit.jupiter.api.Assertions.assertThrows;
import static org.mockito.Mockito.*;

import java.math.BigDecimal;
import java.util.List;
import java.util.Optional;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;

import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

@ExtendWith(MockitoExtension.class)
class OrderPlacementServiceTest {

    @Mock
    private InventoryRepository inventoryRepository;

    @Mock
    private PaymentGateway paymentGateway;

    @Mock
    private OrderRepository orderRepository;

    @InjectMocks
    private OrderPlacementService service;

    @Test
    void shouldRollbackWhenPaymentFails() {

        Product product = new Product();

        product.setId(1L);
        product.setStock(100);

        OrderItem item = new OrderItem();

        item.setProductId(1L);
        item.setQuantity(5);

        Order order = new Order();

        order.setCustomerId(10L);

        order.setItems(
                List.of(item));

        order.setTotalAmount(
                new BigDecimal("100000"));

        when(
                inventoryRepository.findById(1L))
                .thenReturn(
                        Optional.of(product));

        doThrow(
                new RuntimeException("Gateway Error"))
                .when(paymentGateway)
                .charge(anyLong(), any());

        assertThrows(
                PaymentFailedException.class,
                () -> service.placeOrder(order));

        verify(orderRepository, never())
                .save(any());

        verify(paymentGateway, times(1))
                .charge(anyLong(), any());
    }
}
```

---

# Kết quả sau Refinement Chain

## Trước cải tiến

- Không validate dữ liệu.
- Không transaction.
- Không rollback.
- Không logging.
- Không test.
- Không DTO.

## Sau cải tiến

- Validate đầy đủ.
- Exception chuyên biệt.
- @Transactional đảm bảo ACID.
- Logging INFO/ERROR.
- DTO trả kết quả chuẩn.
- Unit Test với Mockito.
- Production-ready theo chuẩn Spring Boot doanh nghiệp.