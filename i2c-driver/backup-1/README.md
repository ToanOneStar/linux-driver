## 1. **Các bước viết driver I2C cơ bản**

### a. Khai báo các thông tin cần thiết
- **Định nghĩa tên thiết bị, địa chỉ I2C, bus I2C**:
  ```c
  #define I2C_BUS_AVAILABLE  (1)
  #define SLAVE_DEVICE_NAME  ("ETX_OLED")
  #define SSD1306_SLAVE_ADDR (0x3C)
  ```
  - Xác định bus I2C sẽ dùng, tên thiết bị, địa chỉ slave.

- **Khai báo các struct quản lý adapter và client**:
  ```c
  static struct i2c_adapter* etx_i2c_adapter = NULL;
  static struct i2c_client* etx_i2c_client_oled = NULL;
  ```
  - `i2c_adapter`: đại diện cho bus I2C vật lý.
  - `i2c_client`: đại diện cho thiết bị slave (ở đây là OLED).

---

### b. Viết các hàm giao tiếp I2C cơ bản

#### **I2C_Write**
```c
static int I2C_Write(unsigned char* buf, unsigned int len)
```
- Gửi dữ liệu qua I2C tới thiết bị slave.
- Sử dụng hàm kernel `i2c_master_send`.
- Trả về số byte gửi thành công hoặc lỗi.

#### **I2C_Read**
```c
static int I2C_Read(unsigned char* out_buf, unsigned int len)
```
- Đọc dữ liệu từ thiết bị slave.
- Sử dụng hàm kernel `i2c_master_recv`.
- Trả về số byte đọc được hoặc lỗi.

---

### c. Viết các hàm điều khiển thiết bị cụ thể

#### **SSD1306_Write**
```c
static void SSD1306_Write(bool is_cmd, unsigned char data)
```
- Gửi lệnh hoặc dữ liệu tới OLED.
- Byte đầu là control byte (0x00 cho lệnh, 0x40 cho data).
- Gọi `I2C_Write` để gửi 2 byte (control + data).

#### **SSD1306_DisplayInit**
```c
static int SSD1306_DisplayInit(void)
```
- Gửi chuỗi lệnh khởi tạo OLED SSD1306.
- Gọi nhiều lần `SSD1306_Write(true, ...)` với các lệnh theo datasheet.

#### **SSD1306_Fill**
```c
static void SSD1306_Fill(unsigned char data)
```
- Gửi dữ liệu lấp đầy toàn bộ màn hình OLED.
- Lặp qua toàn bộ bộ nhớ hiển thị, gửi data.

---

### d. Viết các hàm callback cho driver

#### **etx_oled_probe**
```c
static int etx_oled_probe(struct i2c_client* client)
```
- Được kernel gọi khi phát hiện thiết bị slave phù hợp.
- Thường dùng để khởi tạo thiết bị, cấp phát tài nguyên.
- Ở đây: khởi tạo OLED và fill màn hình sáng toàn bộ.

#### **etx_oled_remove**
```c
static void etx_oled_remove(struct i2c_client* client)
```
- Được kernel gọi khi thiết bị bị tháo ra.
- Thường dùng để giải phóng tài nguyên, tắt thiết bị.
- Ở đây: fill màn hình tắt toàn bộ.

---

### e. Đăng ký thông tin thiết bị và driver với kernel

#### **i2c_device_id**
```c
static const struct i2c_device_id etx_oled_id[] = {{SLAVE_DEVICE_NAME, 0}, {}};
```
- Danh sách các thiết bị mà driver này hỗ trợ.

#### **i2c_driver**
```c
static struct i2c_driver etx_oled_driver = {
    .driver = { .name = SLAVE_DEVICE_NAME, .owner = THIS_MODULE, },
    .probe = etx_oled_probe,
    .remove = etx_oled_remove,
    .id_table = etx_oled_id,
};
```
- Đăng ký các hàm callback, tên driver, bảng id.

#### **i2c_board_info**
```c
static struct i2c_board_info oled_i2c_board_info = {I2C_BOARD_INFO(SLAVE_DEVICE_NAME, SSD1306_SLAVE_ADDR)};
```
- Thông tin về thiết bị slave sẽ được tạo trên bus.

---

### f. Hàm khởi tạo và giải phóng module

#### **etx_driver_init**
```c
static int __init etx_driver_init(void)
```
- Được gọi khi insmod module.
- Lấy adapter, tạo client, đăng ký driver với kernel.

#### **etx_driver_exit**
```c
static void __exit etx_driver_exit(void)
```
- Được gọi khi rmmod module.
- Hủy đăng ký driver, giải phóng client.

---

## 2. **Tóm tắt luồng hoạt động**

1. Khi insmod, hàm `etx_driver_init` chạy:
   - Lấy adapter, tạo client, đăng ký driver.
2. Khi kernel phát hiện thiết bị phù hợp, gọi `etx_oled_probe`:
   - Khởi tạo OLED, fill sáng màn hình.
3. Khi rmmod, hàm `etx_driver_exit` chạy:
   - Hủy đăng ký driver, giải phóng client.
4. Khi thiết bị bị tháo, kernel gọi `etx_oled_remove`:
   - Fill tắt màn hình.

---

## 3. **Tại sao lại làm như vậy?**

- **Tách biệt các hàm giao tiếp I2C**: Để dễ tái sử dụng, bảo trì, và kiểm soát việc gửi/nhận dữ liệu.
- **Hàm probe/remove**: Đúng chuẩn kernel, giúp driver hoạt động tự động khi thiết bị xuất hiện/mất đi.
- **Khởi tạo/giải phóng tài nguyên đúng lúc**: Tránh rò rỉ bộ nhớ, đảm bảo thiết bị hoạt động ổn định.
- **Đăng ký driver và thiết bị**: Để kernel biết driver này điều khiển thiết bị nào, trên bus nào.
