

# BÀI 1: PHÂN BIỆT CÁC LOẠI DELAY

## DELAY (HAL_Delay)
- Là loại delay phổ biến không thuộc HĐH
- Tùy mỗi thư viện sẽ có mỗi cấu trúc khác.
- **Không Block task khi Delay** - Đây là BLOCKING DELAY
- CPU vẫn tiếp tục thực thi trong task hiện tại (busy waiting)
- Nên những task có mức độ ưu tiên thấp hơn không được hoạt động khi những task có mức độ ưu tiên cao hơn đang hoạt động khi dùng delay này (HAL_Delay).
- **Lãng phí tài nguyên CPU** vì không cho phép task khác chạy

Ví dụ như: `HAL_Delay`

### Nhược điểm của HAL_Delay:
- Không tương thích với RTOS scheduler
- Gây ra starvation cho các task khác
- Không tận dụng được khả năng multitasking

## osDelay (vTaskDelay)
- Thuộc hệ điều hành RTOS
- **Khi Delay sẽ Block task** - NON-BLOCKING DELAY
- Task chuyển sang trạng thái BLOCKED
- Scheduler có thể chạy các task khác
- **Relative delay** - thời gian delay tính từ lúc gọi hàm

### Ưu điểm:
- Tiết kiệm tài nguyên CPU
- Cho phép multitasking hiệu quả
- Task tự động chuyển sang READY state khi hết thời gian delay

## osDelayUntil (vTaskDelayUntil)
- Như osDelay
- Khi Delay sẽ Block task. Nhưng khác ở chỗ mốc thời gian Delay.
- **Absolute delay** - thời gian delay tính từ một mốc thời gian cố định
    - Đối với osDelay: Là từ lúc gọi hàm
    - osDelayUntil sẽ là trước khi thời gian gọi hàm. Tức là từ thời điểm mà gán địa chỉ của con trỏ bắt đầu cho tới thời gian truyền vào

### Ưu điểm của osDelayUntil:
- **Periodic execution** - thực thi định kỳ chính xác
- Không bị drift do thời gian xử lý
- Thích hợp cho control loops, sampling data

## So sánh chi tiết:

| Loại Delay | Task State | CPU Usage | Scheduler | Use Case |
|------------|------------|-----------|-----------|----------|
| HAL_Delay | RUNNING | Cao (busy wait) | Không hoạt động | Bare metal, khởi tạo |
| osDelay | BLOCKED | Thấp | Hoạt động | General delay |
| osDelayUntil | BLOCKED | Thấp | Hoạt động | Periodic tasks |

## Các hàm Delay khác trong FreeRTOS:

### 1. vTaskSuspend() / vTaskResume()
- Suspend task vô thời hạn
- Cần task khác để resume
```C
vTaskSuspend(TaskHandle);  // Suspend task
vTaskResume(TaskHandle);   // Resume task
```

### 2. ulTaskNotifyTake() / xTaskNotifyGive()
- Synchronization giữa các task
- Có thể set timeout
```C
// Task 1 - Wait for notification
ulTaskNotifyTake(pdTRUE, portMAX_DELAY);

// Task 2 - Send notification  
xTaskNotifyGive(TaskHandle);
```

### 3. Semaphore với Timeout
```C
xSemaphoreTake(semaphore, pdMS_TO_TICKS(1000)); // Wait 1 second max
```

Demo 1:
```C
void AboveNormalTask(void const *parameter) {
	uint32_t startTime = xTaskGetTickCount();
	while(1){
		printf("From AboveNormalTask \r\n");
		osDelay(200);                    // Block 200ms từ lúc gọi
		osDelayUntil(&startTime, 500);   // Block đến khi đủ 500ms từ startTime
		// Tổng thời gian: 500ms (vì osDelayUntil override osDelay)
	}
}
```

Như trong code Demo 1 thì tổng chạy cũng chỉ mất 500 ms cho dù có 200ms trước đó. 

Demo 2:
```C
void AboveNormalTask(void const *parameter) {
	uint32_t startTime = xTaskGetTickCount();
	while(1){
		printf("From AboveNormalTask \r\n");
		osDelay(200);                    // Block 200ms
		osDelayUntil(&startTime, 100);   // Thời gian đã qua 200ms > 100ms
		// osDelayUntil sẽ return ngay lập tức
		// Tổng thời gian: 200ms
	}
}
```
Trong code Demo 2 trên thì tổng chạy mất 200 ms vì sau khi hàm osDelay được gọi thì đã chạy mất 100ms của osDelayUntil rồi.

## Best Practices:

### 1. Sử dụng osDelayUntil cho Periodic Tasks:
```C
void PeriodicTask(void const *parameter) {
    TickType_t xLastWakeTime = xTaskGetTickCount();
    const TickType_t xFrequency = pdMS_TO_TICKS(100); // 100ms period
    
    while(1) {
        // Do work here
        ProcessData();
        
        // Wait for next cycle - chính xác 100ms
        vTaskDelayUntil(&xLastWakeTime, xFrequency);
    }
}
```

### 2. Sử dụng osDelay cho Non-critical timing:
```C
void SimpleTask(void const *parameter) {
    while(1) {
        // Do work
        SendData();
        
        // Simple delay - không cần timing chính xác
        osDelay(pdMS_TO_TICKS(500));
    }
}
```

### 3. Tránh HAL_Delay trong RTOS:
```C
// ❌ Sai - trong RTOS task
void BadTask(void const *parameter) {
    while(1) {
        DoWork();
        HAL_Delay(1000); // Blocks scheduler!
    }
}

// ✅ Đúng - trong RTOS task  
void GoodTask(void const *parameter) {
    while(1) {
        DoWork();
        osDelay(pdMS_TO_TICKS(1000)); // Allows other tasks
    }
}
```

## Timing Accuracy:

### osDelay - Có drift:
```C
// Mỗi loop có thể bị drift do thời gian ProcessData()
while(1) {
    ProcessData();           // Tốn X ms
    osDelay(100);           // Delay 100ms
    // Tổng: 100 + X ms (drift tích lũy)
}
```

### osDelayUntil - Không drift:
```C
TickType_t xLastWakeTime = xTaskGetTickCount();
while(1) {
    ProcessData();                              // Tốn X ms
    vTaskDelayUntil(&xLastWakeTime, 100);      // Delay để tổng = 100ms
    // Tổng: chính xác 100ms (no drift)
}
```

## Lưu ý về Tick Rate:
- FreeRTOS tick thường là 1000Hz (1ms resolution)
- Delay minimum = 1 tick
- `pdMS_TO_TICKS()` macro chuyển đổi ms sang ticks
```C
// Ví dụ với configTICK_RATE_HZ = 1000
osDelay(pdMS_TO_TICKS(50));  // 50 ticks = 50ms
osDelay(1);                  // 1 tick = 1ms (minimum)
```


# BÀI 2: INTERRUPT TRONG RTOS

## I. Tổng quan về Interrupt trong RTOS

### 1. Interrupt là gì?

* **Interrupt (ngết)** là một cơ chế trong vi điều khiển cho phép CPU **tạm dừng** thực thi chương trình hiện tại để xử lý một sự kiện quan trọng hơn (chẳng hạn như ngắt ngoài, timer, giao tiếp UART...).

### 2. Interrupt trong RTOS là gì?

* Trong RTOS, Interrupt được dùng để:

  * **Gây ra context switch** (chuyển CPU sang task khác)
  * **Unblock task** (lấy task từ BLOCKED sang READY)
  * **Trigger sự kiện** để task xử lý

![Alt text](img/ISR_Visual.png)
---

## II. System Tick trong RTOS

### 1. Tick Interrupt là gì?

* Đây là một **ngết định kỳ do Timer** (thường là SysTick) tạo ra, thường với tần số 1kHz (1ms).
* Mỗi khi ngết Tick xảy ra:

  * RTOS cập nhật System Tick Counter
  * Kiểm tra và chuyển trạng thái các task (delay timeout, unblock,...)
  * Quyết định có context switch hay không

### 2. Đặc tính:

* **Priority không cao nhất**, có thể bị ngắt khác đè
* Trong khi ISR Tick đang chạy, thường **mask các ngắt bằng hoặc thấp hơn**
* Có thể vô hiệu hóa tạm Tick (Tickless mode) để tiết kiệm năng lượng

---

## III. Phân loại Interrupt trong RTOS

### 1. System Tick Interrupt

* Do Timer sinh ra, dùng cho scheduler
* Không do user code tạo
* Cập nhật thời gian hệ thống
* Gây ra context switch nếu cần

### 2. User Interrupt (External IRQ)

* Do ngoại vi sinh ra (nút nhấn, UART, ADC, v.v)
* Có thể tấn công Kernel nếu đặt priority quá cao
* Dùng API FromISR để giao tiếp với task
* Có thể wake up task hoặc gửi dữ liệu qua queue/semaphore

---

## IV. Context Switching trong Interrupt

### 1. Bare Metal (không RTOS):

```c
void EXTI0_IRQHandler(void) {
    ProcessInterrupt(); // xử lý ngay trong ngết
}
```

### 2. Dùng RTOS:

```c
void EXTI0_IRQHandler(void) {
    BaseType_t xHigherPriorityTaskWoken = pdFALSE;

    // Xử lý nhanh
    ReadSensorData();

    // Gửi notification cho task
    vTaskNotifyGiveFromISR(TaskHandle, &xHigherPriorityTaskWoken);

    // Cho phép chuyển task nếu cần
    portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
}
```

---

## V. ISR-Safe API trong FreeRTOS

### ❌ KHÔNG được dùng trong ISR:

* `xTaskCreate()`
* `vTaskDelay()`
* `xQueueReceive()`
* `xSemaphoreTake()`
* `printf()` (không thread-safe)

### ✅ DÙng được trong ISR:

* `xQueueSendFromISR()`
* `xTaskNotifyGiveFromISR()`
* `xSemaphoreGiveFromISR()`
* `portYIELD_FROM_ISR()`

---

## VI. Deferred Interrupt Processing

### 1. Task Notification:

* ISR gửi notification cho task
* Task xử lý sau:

```c
// ISR
vTaskNotifyGiveFromISR(xTaskHandle, &xHigherPriorityTaskWoken);
portYIELD_FROM_ISR(xHigherPriorityTaskWoken);

// Task
ulTaskNotifyTake(pdTRUE, portMAX_DELAY);
```

### 2. Queue From ISR:

```c
xQueueSendFromISR(xQueue, &data, &xHigherPriorityTaskWoken);
```

---

## VII. Cài đặt Interrupt Priority (STM32)

```c
HAL_NVIC_SetPriorityGrouping(NVIC_PRIORITYGROUP_4);
HAL_NVIC_SetPriority(EXTI0_IRQn, 5, 0); // Priority = 5 (số nhỏ hơn = ưu tiên cao hơn)
```

* **Priority 0-4**: Cao, interrupt được phép truy cập trực tiếp
* **Priority 5-15**: An toàn cho RTOS (không gây crash)

---

## VIII. Critical Sections

### 1. Bằng lệnh:

```c
taskENTER_CRITICAL();
// Đoạn code quan trọng
shared++;
taskEXIT_CRITICAL();
```

### 2. Trong ISR:

```c
UBaseType_t state = taskENTER_CRITICAL_FROM_ISR();
// Xử lý
shared++;
taskEXIT_CRITICAL_FROM_ISR(state);
```

### 3. Dùng Mutex:

```c
xSemaphoreTake(xMutex, portMAX_DELAY);
// Vùng gây tranh chấp
xSemaphoreGive(xMutex);
```

---

## IX. Best Practices cho ISR trong RTOS

### ✅ Nên:

* Giữ ISR ngắn, xử lý nhén
* Gửi data cho task qua queue/semaphore/notify
* Dùng các API FromISR

### ❌ Tránh:

* Xử lý phức tạp trong ISR
* Gọi printf, HAL\_Delay, malloc...
* Dùng các API không FromISR

---

## X. Debug Interrupt

### 1. Các lỗi hay gặp:

* **Hard Fault**: Dùng sai API
* **Stack Overflow**: ISR stack không đủ
* **Priority inversion**: Đặt sai ngưỡng

### 2. Debug GPIO:

```c
GPIO_Set(DEBUG_PIN);
// ISR Code
GPIO_Reset(DEBUG_PIN);
```

---

## XI. Kết luận

* Interrupt trong RTOS phục vụ cho scheduling, event notify, giao tiếp nhanh
* Nên giữ ISR đơn giản, dùng API FromISR
* Đặt đúng priority để không crash hệ thống
* RTOS đảm bảo context switching mềm dẻ nhưng cần tuân thủ nghiêm ngặt quy tắc ISR
