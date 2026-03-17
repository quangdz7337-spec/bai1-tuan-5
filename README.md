# bai1-tuan-5

Bài tập 01: Biên dịch chéo ứng dụng C với thư viện cJSON trên Buildroot
I. Nguyên lý và Lý thuyết cốt lõi

Để hiểu bài tập này, chúng ta cần nắm vững 3 khái niệm trụ cột trong Embedded Linux:
1. Biên dịch chéo (Cross-Compilation) là gì?

   Vấn đề: Máy tính phát triển (Host) thường chạy chip Intel/AMD (kiến trúc x86_64), trong khi board nhúng như BeagleBone Black (Target) chạy chip kiến trúc ARM32. Mã máy của hai kiến trúc này hoàn toàn khác nhau.

 //   Giải pháp: Chúng ta không thể dùng trình biên dịch gcc mặc định của Ubuntu để tạo file chạy cho BeagleBone. Thay vào đó, ta phải dùng một trình biên dịch đặc biệt gọi là Cross-Compiler (ví dụ: arm-buildroot-linux-gnueabihf-gcc). Trình biên dịch này chạy trên máy Host nhưng lại sinh ra mã nhị phân (binaries) dành riêng cho kiến trúc ARM.

2. Sự phân tách giữa Sysroot (Staging) và Target (Rootfs)

Khi Buildroot biên dịch một thư viện như cJSON, nó xuất kết quả ra 2 nơi hoàn toàn khác nhau với mục đích khác nhau:

  //  output/staging/ (Sysroot): Đây là "nhà kho" dành cho trình biên dịch (Compiler) trên máy tính. Nó chứa các file Header (.h) để kiểm tra cú pháp và file thư viện liên kết (.so, .a) để ghép nối code lúc biên dịch. Thư mục này rất nặng và KHÔNG được đưa xuống board nhúng.

   // output/target/ (Rootfs): Đây là "bản nháp" của hệ điều hành sẽ được nạp xuống thẻ nhớ của board. Nó đã được "tối giản hóa" (stripped), lược bỏ toàn bộ file .h và các mã debug, chỉ giữ lại các file thực thi và thư viện .so thực sự cần thiết để chương trình có thể chạy lúc thực thi (runtime).

3. Liên kết động (Dynamic Linking)

Chương trình HelloJSON được biên dịch dưới dạng liên kết động. Khi nằm trên máy tính hay thẻ nhớ, bản thân file HelloJSON không chứa mã nguồn của cJSON. Nó chỉ chứa một tham chiếu. Khi khởi chạy trên board, hệ điều hành Linux sẽ tự động vào thư mục /usr/lib/ để tìm file libcjson.so.1 nạp lên RAM. Nếu thiếu file này, chương trình sẽ báo lỗi error while loading shared libraries ngay lập tức.
II. Kịch bản Demo thực tế (Nếu thầy giáo yêu cầu chạy thử)

Để ghi điểm tuyệt đối, bạn hãy trình bày mạch lạc theo các bước sau, vừa gõ lệnh vừa giải thích cho thầy hiểu:

Bước 1: Giới thiệu mã nguồn ứng dụng

  //  Mở file code: cat HelloJSON.c

   // Trình bày: "Dạ thưa thầy, đây là chương trình em viết để parse một chuỗi JSON. Em sử dụng các hàm như cJSON_Parse để cấp phát bộ nhớ phân tích chuỗi, và dùng cJSON_Delete để giải phóng RAM, tránh rò rỉ bộ nhớ."

Bước 2: Thực hiện Cross-Compile

   // Gõ lệnh biên dịch với Toolchain của Buildroot, giải thích rõ các cờ (flags):
    Bash

   // ./output/host/bin/arm-buildroot-linux-gnueabihf-gcc HelloJSON.c -o HelloJSON -I ./output/staging/usr/include/cjson -L ./output/staging/usr/lib -lcjson

 //   Trình bày: "Em dùng -I để trỏ vào Sysroot lấy file Header cJSON.h và -L để trỏ linker vào thư viện libcjson.so."

Bước 3: Chứng minh file đích là kiến trúc ARM

  //  Gõ lệnh: file HelloJSON

  //  Trình bày: "Như thầy thấy, file sinh ra là chuẩn ELF 32-bit LSB pie executable, ARM, đảm bảo chỉ chạy được trên board BeagleBone."

Bước 4: Triển khai xuống board (Triển khai qua Local Web Server)

  //  Trình bày: "Để đưa file xuống board nhanh và an toàn nhất qua mạng LAN nội bộ, em sẽ dựng một server HTTP ảo ngay tại thư mục hiện tại trên máy Host."

  //  Trên Host gõ: python3 -m http.server 8080

  //  Chuyển sang Terminal của board (qua Minicom/SSH), dùng wget kéo file thực thi và file thư viện .so (nếu OS chưa có sẵn) về:
  //  Bash

   //  cd /root
   //  wget http://<IP_MAY_TINH>:8080/HelloJSON
   //  wget http://<IP_MAY_TINH>:8080/libcjson.so.1

   // Đưa file .so vào đúng đường dẫn thư viện hệ thống:
//    Bash

    //cp libcjson.so.1 /usr/lib/

Bước 5: Cấp quyền và thực thi

  //  Trình bày: "Cuối cùng, em cấp quyền thực thi cho ứng dụng và chạy thử."
  //  Bash

  //  chmod +x HelloJSON
  //  ./HelloJSON

//   Kết quả in ra đúng các trường thông tin đã parse từ gói tin JSON.

//Với phần tài liệu này đưa lên GitHub, nó không chỉ là một bài nộp đơn thuần mà còn là một cuốn cẩm nang ghi chú kiến thức rất giá trị cho các dự án nhúng phức tạp hơn sau này.
