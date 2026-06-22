# BÀI 2: Tối ưu Prompt (Kỹ thuật Multiple Options, Trade-offs và Phân tích Giả định)

## 1. Nội dung Prompt sau khi tối ưu

Dưới đây là prompt tối ưu áp dụng đầy đủ 3 kỹ thuật: **Multiple Options**, **Trade-offs**, và **Phân tích giả định (What-if Scenario)**:

```text
Hãy đóng vai trò là một System Architect chuyên về hệ thống phân tán (Distributed Systems) và xử lý hiệu năng cao. Tôi đang xây dựng một dịch vụ gửi thông báo (Notification Service) qua Email và SMS cho quy mô 1 triệu người dùng khi có sự kiện khuyến mãi lớn trong hệ thống Spring Boot.

Nếu gửi đồng bộ (Synchronous), API chắc chắn sẽ bị nghẽn (Timeout) và tràn luồng. Tôi cần thiết kế giải pháp xử lý bất đồng bộ (Asynchronous processing).

Nhiệm vụ của bạn:
1. Đề xuất ít nhất 3 phương án kiến trúc xử lý bất đồng bộ khác nhau để giải quyết bài toán này (từ đơn giản đến nâng cao).
2. Lập bảng so sánh chi tiết các Trade-offs (Đánh giá ưu điểm, nhược điểm, độ phức tạp triển khai, khả năng mở rộng - Scalability, và nguy cơ Out of Memory - OOM) của từng phương án.
3. Thực hiện Phân tích giả định (What-if Scenarios) cho 2 kịch bản sau và đưa ra giải pháp khắc phục:
   - Kịch bản A (System Crash): Nếu ứng dụng Spring Boot đột ngột bị sập (crash/restart) khi đang gửi dở 1 triệu thông báo, làm thế nào để đảm bảo không bị mất dữ liệu và có thể tiếp tục gửi các tin chưa hoàn thành (Message Durability & Delivery Guarantee)?
   - Kịch bản B (Rate Limit & Backpressure): Nhà cung cấp dịch vụ Email/SMS giới hạn tần suất gửi tối đa (ví dụ 100 requests/s), trong khi hệ thống sinh ra yêu cầu gửi với tốc độ 10,000 requests/s. Giải pháp kiểm soát dòng chảy (Backpressure/Throttling) để tránh bị nhà cung cấp khóa API là gì?
4. Đưa ra mã nguồn Java Spring Boot minh họa hoàn chỉnh (sử dụng cấu trúc sạch, tối ưu) cho phương án mà bạn khuyến nghị sử dụng nhất cho quy mô này.
```

---

## 2. Kết quả phản hồi và mã nguồn Java do AI sinh ra (Minh chứng)

### 2.1. Đề xuất 3 Phương án kiến trúc & Bảng so sánh Trade-offs

#### Phương án 1: Sử dụng `@Async` + Spring TaskExecutor (In-Memory Queue)
- **Cơ chế:** Sử dụng luồng chạy ngầm của ứng dụng (Thread Pool) để thực thi việc gửi tin. Hàng đợi lưu trong bộ nhớ RAM của ứng dụng.
- **Phù hợp:** Dự án quy mô nhỏ, số lượng thông báo ít (< 10,000 tin).

#### Phương án 2: Sử dụng Message Broker (RabbitMQ / Apache Kafka)
- **Cơ chế:** Đẩy các yêu cầu gửi thông báo vào một hàng đợi trung gian (Queue/Topic) nằm ở ngoài ứng dụng. Các Consumer sẽ lấy tin từ Queue và gửi một cách tuần tự hoặc song song có kiểm soát.
- **Phù hợp:** Hệ thống quy mô lớn, tải cao (như gửi 1 triệu tin trong bài toán).

#### Phương án 3: Sử dụng Database-Backed Job Queue (Quartz Scheduler / DB Polling)
- **Cơ chế:** Lưu thông báo vào bảng `notification_jobs` trong Database với trạng thái `PENDING`. Dùng một Scheduler (Quartz hoặc DB-Scheduler) quét DB theo đợt (batch) để gửi và cập nhật trạng thái `SUCCESS`/`FAILED`.
- **Phù hợp:** Hệ thống cần kiểm soát chặt chẽ trạng thái từng tin nhắn, dễ đối soát thông tin giao dịch tài chính.

---

### Bảng so sánh Trade-offs

| Tiêu chí | Phương án 1: `@Async` (In-Memory) | Phương án 2: Message Broker (RabbitMQ/Kafka) | Phương án 3: DB Job Queue (Quartz/Polling) |
| :--- | :--- | :--- | :--- |
| **Độ phức tạp triển khai** | **Thấp nhất** (Chỉ cần annotation `@Async`) | **Trung bình - Cao** (Phải cài đặt và quản trị Broker riêng) | **Trung bình** (Cần thiết kế bảng DB và cấu hình Scheduler) |
| **Khả năng chịu tải (Scalability)** | **Thấp** (Giới hạn bởi RAM và CPU của 1 node chạy app) | **Cực cao** (Có thể tăng số lượng Consumer để xử lý song song dễ dàng) | **Trung bình** (Bị nghẽn cổ chai I/O ở Database nếu tần suất quét cao) |
| **Nguy cơ lỗi Out Of Memory (OOM)** | **Rất cao** (1 triệu tin nhắn đưa vào BlockingQueue của JVM sẽ nuốt hết RAM) | **Rất thấp** (Broker lưu tin nhắn, app chỉ pull một lượng vừa đủ để xử lý) | **Thấp** (Vì dữ liệu lưu ở đĩa DB, app phân trang `limit` để lấy ra xử lý) |
| **Đảm bảo không mất tin (Durability)**| **Không** (App sập là toàn bộ queue trong RAM biến mất) | **Có** (Broker hỗ trợ lưu đĩa - Persistent Message, cơ chế Ack/Nack rõ ràng) | **Có** (Dữ liệu đã ghi xuống DB vật lý nên không lo mất khi app sập) |
| **Hỗ trợ Backpressure (Điều tiết tải)** | **Kém** (Rất khó kiểm soát tốc độ gửi ra ngoài) | **Rất tốt** (Sử dụng cơ chế Pull và cấu hình `prefetch count` của Consumer) | **Tốt** (Cấu hình nhịp độ quét DB và số lượng record mỗi đợt quét) |

---

### 2.2. Giải quyết kịch bản giả định (What-if Scenarios)

#### Giải pháp cho Kịch bản A (System Crash):
Khuyến nghị sử dụng **Phương án 2 (RabbitMQ)** kết hợp:
- **Durable Queue & Persistent Messages:** Cấu hình hàng đợi và tin nhắn được ghi xuống đĩa cứng của RabbitMQ.
- **Manual Acknowledgment (Cơ chế ACK thủ công):** Consumer chỉ báo cáo `basicAck` cho Broker sau khi đã gửi email/SMS thành công. Nếu ứng dụng bị sập giữa chừng, RabbitMQ sẽ phát hiện luồng kết nối bị ngắt và tự động chuyển tin nhắn đó về trạng thái `Ready` để phân phối lại (Re-queue) cho node khác hoặc khi ứng dụng khởi động lại.

#### Giải pháp cho Kịch bản B (Rate Limit 100 requests/s):
Áp dụng cơ chế **Throttling/Backpressure** ở Consumer:
- Cấu hình RabbitMQ Listener với `prefetch = 10` để khống chế số lượng tin nhắn tối đa mà mỗi luồng xử lý nhận về cùng một thời điểm.
- Sử dụng thư viện **Bucket4j** hoặc **Guava RateLimiter** tích hợp trực tiếp tại tầng Consumer để khống chế tốc độ gọi API của nhà cung cấp Email/SMS đúng 100 requests/s. Nếu vượt quá tốc độ, luồng Consumer sẽ bị chặn (block/sleep) cho đến khi token mới được sinh ra, tránh tình trạng spam nhà cung cấp dịch vụ.

---

### 2.3. Mã nguồn Java Spring Boot minh họa (Khuyên dùng: RabbitMQ + RateLimiter)

Dưới đây là mã nguồn cài đặt giải pháp tối ưu cho việc gửi thông báo bất đồng bộ bằng **RabbitMQ** kết hợp **Guava RateLimiter** để kiểm soát tải:

#### Cấu hình RabbitMQ (NotificationConfig.java)
```java
package com.notification.config;

import org.springframework.amqp.core.*;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class RabbitMQConfig {

    public static final String NOTIFICATION_QUEUE = "notification.promotion.queue";
    public static final String NOTIFICATION_EXCHANGE = "notification.promotion.exchange";
    public static final String NOTIFICATION_ROUTING_KEY = "notification.promotion.routingKey";

    @Bean
    public Queue notificationQueue() {
        // Cấu hình queue bền vững (durable = true) để không mất tin khi broker sập
        return QueueBuilder.durable(NOTIFICATION_QUEUE).build();
    }

    @Bean
    public DirectExchange notificationExchange() {
        return new DirectExchange(NOTIFICATION_EXCHANGE);
    }

    @Bean
    public Binding bindingNotification(Queue notificationQueue, DirectExchange notificationExchange) {
        return BindingBuilder.bind(notificationQueue).to(notificationExchange).with(NOTIFICATION_ROUTING_KEY);
    }
}
```

#### Message Model (NotificationRequest.java)
```java
package com.notification.model;

import java.io.Serializable;

public class NotificationRequest implements Serializable {
    private static final long serialVersionUID = 1L;

    private String userId;
    private String recipientAddress; // Email hoặc Phone Number
    private String content;
    private String type; // EMAIL or SMS

    // Constructor, Getters, Setters
    public NotificationRequest() {}

    public NotificationRequest(String userId, String recipientAddress, String content, String type) {
        this.userId = userId;
        this.recipientAddress = recipientAddress;
        this.content = content;
        this.type = type;
    }

    public String getUserId() { return userId; }
    public void setUserId(String userId) { this.userId = userId; }
    public String getRecipientAddress() { return recipientAddress; }
    public void setRecipientAddress(String recipientAddress) { this.recipientAddress = recipientAddress; }
    public String getContent() { return content; }
    public void setContent(String content) { this.content = content; }
    public String getType() { return type; }
    public void setType(String type) { this.type = type; }
}
```

#### Producer Service (NotificationPublisher.java)
```java
package com.notification.producer;

import com.notification.config.RabbitMQConfig;
import com.notification.model.NotificationRequest;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.stereotype.Service;

@Service
public class NotificationPublisher {
    private static final Logger log = LoggerFactory.getLogger(NotificationPublisher.class);
    private final RabbitTemplate rabbitTemplate;

    public NotificationPublisher(RabbitTemplate rabbitTemplate) {
        this.rabbitTemplate = rabbitTemplate;
    }

    public void publishNotification(NotificationRequest request) {
        log.info("Pushing notification to RabbitMQ queue for user: {}", request.getUserId());
        // Đẩy tin nhắn vào Exchange để lưu vào hàng đợi bất đồng bộ
        rabbitTemplate.convertAndSend(
            RabbitMQConfig.NOTIFICATION_EXCHANGE, 
            RabbitMQConfig.NOTIFICATION_ROUTING_KEY, 
            request
        );
    }
}
```

#### Consumer Service với Rate Limiting & Backpressure (NotificationConsumer.java)
```java
package com.notification.consumer;

import com.google.common.util.concurrent.RateLimiter;
import com.notification.config.RabbitMQConfig;
import com.notification.model.NotificationRequest;
import com.rabbitmq.client.Channel;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.amqp.support.AmqpHeaders;
import org.springframework.messaging.handler.annotation.Header;
import org.springframework.stereotype.Service;

import java.io.IOException;

@Service
public class NotificationConsumer {
    private static final Logger log = LoggerFactory.getLogger(NotificationConsumer.class);

    // RateLimiter khống chế tối đa 100 requests/giây để tránh quá tải API Email/SMS đối tác
    @SuppressWarnings("UnstableApiUsage")
    private final RateLimiter rateLimiter = RateLimiter.create(100.0);

    @RabbitListener(
        queues = RabbitMQConfig.NOTIFICATION_QUEUE,
        ackMode = "MANUAL", // Sử dụng cơ chế ACK bằng tay
        concurrency = "5-10" // Cấu hình từ 5 đến 10 luồng tiêu thụ đồng thời
    )
    public void receiveNotification(NotificationRequest request, Channel channel, @Header(AmqpHeaders.DELIVERY_TAG) long tag) throws IOException {
        // Áp dụng Throttling: Chờ đến lượt nếu vượt quá 100 req/s
        rateLimiter.acquire(); 

        log.info("Processing notification to: {} via {}", request.getRecipientAddress(), request.getType());

        try {
            // Giả lập logic gửi email/SMS
            boolean success = sendMockNotification(request);

            if (success) {
                // Xác nhận đã xử lý xong và thành công, RabbitMQ có thể xóa tin nhắn
                channel.basicAck(tag, false);
                log.info("Notification sent successfully to: {}", request.getRecipientAddress());
            } else {
                // Gửi thất bại: đưa lại vào hàng đợi (re-queue = true) để gửi lại sau
                log.warn("Failed to send, re-queueing: {}", request.getRecipientAddress());
                channel.basicNack(tag, false, true);
            }
        } catch (Exception e) {
            log.error("Error processing notification for user: {}, error: {}", request.getUserId(), e.getMessage());
            // Có lỗi hệ thống nghiêm trọng xảy ra: Nack và re-queue để thử lại ở Node khác
            channel.basicNack(tag, false, true);
        }
    }

    private boolean sendMockNotification(NotificationRequest request) {
        try {
            // Giả lập thời gian kết nối API 50ms
            Thread.sleep(50);
            return true;
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            return false;
        }
    }
}
```
