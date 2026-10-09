# [Thực hành] Gọi MySQL Stored Procedures từ JDBC

**Hoàn thành**

## Mục tiêu:

Luyện tập sử dụng Stored Procedures trong MySQL và cách gọi chúng từ ứng dụng Java thông qua CallableStatement của thư viện JDBC.

## Mô tả:

Trong bài thực hành này, chúng ta sẽ cập nhật ứng dụng WEB quản lý User đã xây dựng ở các bài trước. Thay vì sử dụng câu lệnh SQL trực tiếp (PreparedStatement) như cũ, bạn sẽ chuyển sang sử dụng Stored Procedures cho hai chức năng cốt lõi:

- Tìm User theo ID.
- Thêm mới User.

## Hướng dẫn:

### Bước 1: Định nghĩa Stored Procedures trong cơ sở dữ liệu MySQL

Đầu tiên, bạn cần tạo các Stored Procedures trong cơ sở dữ liệu `demo` đang chứa bảng `users`. Mở công cụ quản trị MySQL (như MySQL Workbench, DBeaver, hoặc Terminal) và thực thi đoạn mã SQL sau:

```sql
USE demo;
-- 1. Tạo Stored Procedure để lấy thông tin User theo ID
DELIMITER $$
CREATE PROCEDURE get_user_by_id(IN user_id INT)
BEGIN
    SELECT users.name, users.email, users.country
    FROM users
    WHERE users.id = user_id;
END$$
DELIMITER ;

-- 2. Tạo Stored Procedure để thêm mới một User
DELIMITER $$
CREATE PROCEDURE insert_user(
    IN user_name VARCHAR(50),
    IN user_email VARCHAR(50),
    IN user_country VARCHAR(50)
)
BEGIN
    INSERT INTO users(name, email, country)
    VALUES(user_name, user_email, user_country);
END$$
DELIMITER ;
```

**Lưu ý:** Chú ý từ khóa `DELIMITER` dùng để thay đổi dấu kết thúc câu lệnh mặc định của MySQL, giúp hệ thống không bị lỗi khi biên dịch khối lệnh `BEGIN...END`.

### Bước 2: Cập nhật Interface IUserDAO

Quay lại IDE của bạn (VS Code/IntelliJ/Eclipse), mở file `IUserDAO.java` nằm trong thư mục `src/main/java/com/codegym/dao/`. Bổ sung thêm phần khai báo cho 2 phương thức mới:

```java
package com.codegym.dao;

import com.codegym.model.User;
import java.sql.SQLException;
import java.util.List;

public interface IUserDAO {
    // Các phương thức cũ đã có...
    public void insertUser(User user) throws SQLException;
    public User selectUser(int id);
    public List<User> selectAllUsers();
    public boolean deleteUser(int id) throws SQLException;
    public boolean updateUser(User user) throws SQLException;

    // THÊM MỚI 2 PHƯƠNG THỨC SỬ DỤNG STORED PROCEDURE
    public User getUserById(int id);
    public void insertUserStore(User user) throws SQLException;
}
```

### Bước 3: Cập nhật lớp UserDAO (Sử dụng CallableStatement)

Tiếp theo, mở tệp `UserDAO.java` và triển khai (override) 2 phương thức vừa khai báo. Tại đây, chúng ta sẽ sử dụng `CallableStatement` kết hợp với cú pháp `{CALL procedure_name(?, ?, ?)}` để gọi các thủ tục từ MySQL.

```java
package com.codegym.dao;

import com.codegym.model.User;
import java.sql.*;
import java.util.ArrayList;
import java.util.List;

public class UserDAO implements IUserDAO {
    // Khai báo các thông tin kết nối và phương thức cũ giữ nguyên...
    
    // Triển khai hàm gọi Stored Procedure: get_user_by_id
    @Override
    public User getUserById(int id) {
        User user = null;
        String query = "{CALL get_user_by_id(?)}"; // Cú pháp chuẩn gọi Procedure

        try (Connection connection = getConnection();
             CallableStatement callableStatement = connection.prepareCall(query)) {
            
            callableStatement.setInt(1, id);
            ResultSet rs = callableStatement.executeQuery();

            while (rs.next()) {
                String name = rs.getString("name");
                String email = rs.getString("email");
                String country = rs.getString("country");
                user = new User(id, name, email, country);
            }
        } catch (SQLException e) {
            e.printStackTrace();
        }
        return user;
    }

    // Triển khai hàm gọi Stored Procedure: insert_user
    @Override
    public void insertUserStore(User user) throws SQLException {
        String query = "{CALL insert_user(?, ?, ?)}"; // 3 dấu ? tương ứng 3 tham số IN

        try (Connection connection = getConnection();
             CallableStatement callableStatement = connection.prepareCall(query)) {
            
            callableStatement.setString(1, user.getName());
            callableStatement.setString(2, user.getEmail());
            callableStatement.setString(3, user.getCountry());
            System.out.println(callableStatement);
            callableStatement.executeUpdate();
        }
    }
}
```

### Bước 4: Cập nhật lớp UserServlet (Tầng Controller)

Bây giờ, chúng ta cần thay đổi logic điều hướng tại Controller để ứng dụng sử dụng các phương thức mới thay vì phương thức cũ. Mở tệp `UserServlet.java` và tiến hành sửa đổi.

**1. Sửa đổi chức năng hiển thị form Edit (`showEditForm`):** Thay thế lệnh gọi `userDAO.selectUser(id)` bằng `userDAO.getUserById(id)`.

```java
private void showEditForm(HttpServletRequest request, HttpServletResponse response)
        throws SQLException, ServletException, IOException {
    int id = Integer.parseInt(request.getParameter("id"));
    
    // Sử dụng phương thức mới gọi Stored Procedure
    User existingUser = userDAO.getUserById(id); 
    
    RequestDispatcher dispatcher = request.getRequestDispatcher("user/edit.jsp");
    request.setAttribute("user", existingUser);
    dispatcher.forward(request, response);
}
```

**2. Sửa đổi chức năng thêm mới người dùng (`insertUser`):** Thay thế lệnh gọi `userDAO.insertUser(newUser)` bằng `userDAO.insertUserStore(newUser)`.

```java
private void insertUser(HttpServletRequest request, HttpServletResponse response)
        throws SQLException, IOException, ServletException {
    String name = request.getParameter("name");
    String email = request.getParameter("email");
    String country = request.getParameter("country");
    User newUser = new User(name, email, country);
    
    // Sử dụng phương thức mới gọi Stored Procedure
    userDAO.insertUserStore(newUser);
    // Áp dụng PRG pattern (sendRedirect) sau khi thêm mới thành công để tránh submit lặp khi F5
    response.sendRedirect("users");
}
```

### Bước 5: Biên dịch, triển khai và kiểm thử

**1. Biên dịch dự án:**

Mở Terminal và gõ lệnh: `mvn clean package` để Maven build lại file `.war` mới nhất bao gồm các cập nhật code vừa thực hiện.

**2. Chạy lại ứng dụng trên Tomcat:**

Nếu Tomcat đang chạy, hãy Restart lại Server hoặc nhấp chuột phải chọn Redeploy tệp `.war` mới.

**3. Quan sát kết quả:**

- Truy cập `http://localhost:8080/user-management/users` (hoặc đường dẫn ánh xạ tương ứng của bạn).
- Nhấn nút "Thêm mới User", điền thông tin và lưu lại. Ứng dụng sẽ gọi thành công `insert_user` procedure.
- Nhấn nút "Edit" trên một User bất kỳ. Ứng dụng sẽ gọi thành công `get_user_by_id` procedure và hiển thị đầy đủ thông tin User cũ lên biểu mẫu.

---

## Prompt ra lệnh cho AI Agent triển khai dự án

**Chuẩn bị trước:** Hãy đảm bảo bạn đã biết chính xác đường dẫn thư mục cài đặt Tomcat trên máy tính (Ví dụ: `C:\Tomcat 10.1` trên Windows hoặc `/Library/Tomcat` trên macOS).

Copy đoạn lệnh dưới đây và dán vào khung chat của AI Agent:

```
Đóng vai là một chuyên gia DevOps, hãy giúp tôi tự động hóa quá trình đóng gói và triển khai dự án Java Web hiện tại lên máy chủ Tomcat.

Yêu cầu thực thi:

1. Biên dịch dự án: Mở terminal tại thư mục gốc của dự án này và chạy lệnh mvn clean package để dọn dẹp và đóng gói dự án. Hãy chờ cho đến khi xuất hiện thông báo BUILD SUCCESS.

2. Xác định file: Tìm file .war vừa được tạo ra bên trong thư mục target/ của dự án.

3. Triển khai (Deploy): Sử dụng lệnh hệ thống (terminal/bash/powershell) để copy file .war này và dán vào thư mục webapps của máy chủ Tomcat tại đường dẫn tuyệt đối sau: [ĐIỀN_ĐƯỜNG_DẪN_THƯ_MỤC_TOMCAT_CỦA_BẠN_VÀO_ĐÂY]

4. Khởi động Server: Sau khi copy thành công, hãy điều hướng terminal tới thư mục bin của Tomcat và chạy file khởi động (startup.bat cho Windows hoặc ./startup.sh cho macOS/Linux).

Hãy hiển thị cho tôi các lệnh bạn sẽ chạy trước khi thực thi để tôi phê duyệt.
```
