Bài tập 01: Biên dịch chéo ứng dụng C với thư viện cJSON trên Buildroot
I. Nguyên lý và Lý thuyết cốt lõi

Để hiểu bài tập này, chúng ta cần nắm vững 3 khái niệm trụ cột trong Embedded Linux:
1. Biên dịch chéo (Cross-Compilation) là gì?

    Vấn đề:
    Máy tính phát triển (Host) thường chạy chip Intel/AMD (kiến trúc x86_64),
   trong khi board nhúng như BeagleBone Black (Target) chạy chip kiến trúc ARM32.
   Mã máy của hai kiến trúc này hoàn toàn khác nhau.

    Giải pháp:
    Chúng ta không thể dùng trình biên dịch gcc mặc định của Ubuntu để tạo file chạy cho BeagleBone.
   Thay vào đó, ta phải dùng một trình biên dịch đặc biệt gọi là Cross-Compiler (ví dụ: arm-buildroot-linux-gnueabihf-gcc).
    Trình biên dịch này chạy trên máy Host nhưng lại sinh ra mã nhị phân (binaries) dành riêng cho kiến trúc ARM.

3. Sự phân tách giữa Sysroot (Staging) và Target (Rootfs)

Khi Buildroot biên dịch một thư viện như cJSON, nó xuất kết quả ra 2 nơi hoàn toàn khác nhau với mục đích khác nhau:

    output/staging/ (Sysroot):
    Đây là "nhà kho" dành cho trình biên dịch (Compiler) trên máy tính.
    Nó chứa các file Header (.h) để kiểm tra cú pháp và file thư viện liên kết (.so, .a) để ghép nối code lúc biên dịch. 
    Thư mục này rất nặng và KHÔNG được đưa xuống board nhúng.

    output/target/ (Rootfs): Đây là "bản nháp" của hệ điều hành sẽ được nạp xuống thẻ nhớ của board.
    Nó đã được "tối giản hóa" (stripped), lược bỏ toàn bộ file .h và các mã debug, chỉ giữ lại các file thực thi và thư viện 
    .so thực sự cần thiết để chương trình có thể chạy lúc thực thi (runtime).

3. Liên kết động (Dynamic Linking)

Chương trình HelloJSON được biên dịch dưới dạng liên kết động. Khi nằm trên máy tính hay thẻ nhớ,
bản thân file HelloJSON không chứa mã nguồn của cJSON. Nó chỉ chứa một tham chiếu.
Khi khởi chạy trên board, hệ điều hành Linux sẽ tự động vào thư mục /usr/lib/ để tìm file libcjson.so.1 nạp lên RAM.
Nếu thiếu file này, chương trình sẽ báo lỗi error while loading shared libraries ngay lập tức.
II. Kịch bản Demo thực tế (Nếu thầy giáo yêu cầu chạy thử)

Để ghi điểm tuyệt đối, bạn hãy trình bày mạch lạc theo các bước sau, vừa gõ lệnh vừa giải thích cho thầy hiểu:

Bước 1: Giới thiệu mã nguồn ứng dụng

    Mở file code: cat HelloJSON.c

    Trình bày: "Dạ thưa thầy, đây là chương trình em viết để parse một chuỗi JSON.
    Em sử dụng các hàm như cJSON_Parse để cấp phát bộ nhớ phân tích chuỗi,
    và dùng cJSON_Delete để giải phóng RAM, tránh rò rỉ bộ nhớ."

Bước 2: Thực hiện Cross-Compile

    Gõ lệnh biên dịch với Toolchain của Buildroot, giải thích rõ các cờ (flags):
    Bash

    ./output/host/bin/arm-buildroot-linux-gnueabihf-gcc HelloJSON.c 
    -o HelloJSON 
    -I ./output/staging/usr/include/cjson 
    -L ./output/staging/usr/lib 
    -lcjson

    Trình bày: "Em dùng -I để trỏ vào Sysroot lấy file Header cJSON.h và -L để trỏ linker vào thư viện libcjson.so."

Bước 3: Chứng minh file đích là kiến trúc ARM

    Gõ lệnh: file HelloJSON

    Trình bày: "Như thầy thấy, file sinh ra là chuẩn ELF 32-bit LSB pie executable, ARM, đảm bảo chỉ chạy được trên board BeagleBone."

Bước 4: Triển khai xuống board (Triển khai qua Local Web Server)

    Trình bày:
    "Để đưa file xuống board nhanh và an toàn nhất qua mạng LAN nội bộ,
    em sẽ dựng một server HTTP ảo ngay tại thư mục hiện tại trên máy Host."

    Trên Host gõ: python3 -m http.server 8080

    Chuyển sang Terminal của board (qua Minicom/SSH), dùng wget kéo file thực thi và file thư viện .so (nếu OS chưa có sẵn) về:
    Bash

    cd /root
    wget http://<IP_MAY_TINH>:8080/HelloJSON
    wget http://<IP_MAY_TINH>:8080/libcjson.so.1

    Đưa file .so vào đúng đường dẫn thư viện hệ thống:
    Bash

    cp libcjson.so.1 /usr/lib/

Bước 5: Cấp quyền và thực thi

    Trình bày: "Cuối cùng, em cấp quyền thực thi cho ứng dụng và chạy thử."
    Bash

    chmod +x HelloJSON
    ./HelloJSON

    Kết quả in ra đúng các trường thông tin đã parse từ gói tin JSON.

Với phần tài liệu này đưa lên GitHub, nó không chỉ là một bài nộp đơn thuần mà còn là một cuốn cẩm nang ghi chú kiến thức rất giá trị cho các dự án nhúng phức tạp hơn sau này.


Bài tập 02: Tự tạo và sử dụng Thư viện tĩnh (.a) & Thư viện động (.so) trên Embedded Linux
I. Nguyên lý và Lý thuyết cốt lõi

Trong môi trường lập trình nhúng (Embedded Linux),
việc tối ưu hóa bộ nhớ và quản lý mã nguồn là vô cùng quan trọng. 
Bài tập này giúp làm rõ hai phương pháp liên kết thư viện cơ bản khi biên dịch chéo cho board BeagleBone Black (kiến trúc ARM).
1. Thư viện Tĩnh (Static Library - .a)

    Bản chất: Là một tập hợp các file object (.o) được nén lại (Archive). Khi biên dịch, Trình liên kết (Linker) sẽ "mở" thư viện này ra, trích xuất mã máy của các hàm được gọi và sao chép trực tiếp (nhúng) chúng vào bên trong file thực thi cuối cùng.

    Ưu điểm: File thực thi hoạt động hoàn toàn độc lập (standalone). Mang sang bất kỳ board nhúng nào cùng kiến trúc ARM cũng chạy được ngay lập tức mà không sợ lỗi thiếu môi trường.

    Nhược điểm: Dung lượng file thực thi phình to. Nếu có nhiều ứng dụng cùng gọi một thư viện tĩnh, mã nguồn của thư viện đó sẽ bị nhân bản nhiều lần trên RAM và bộ nhớ lưu trữ, gây lãng phí tài nguyên.

2. Thư viện Động (Shared/Dynamic Library - .so)

    Bản chất: Khác với thư viện tĩnh, mã nguồn của thư viện động không được nhúng vào chương trình. Trình liên kết chỉ khắc một "lời nhắc" (reference) vào file thực thi, báo cho hệ điều hành biết rằng: "Khi chạy chương trình này, hãy nạp file .so này vào RAM".

    Ưu điểm: Kích thước file thực thi cực kỳ nhỏ. Nhiều ứng dụng khác nhau có thể dùng chung một file thư viện .so duy nhất trên RAM, giúp tiết kiệm tối đa bộ nhớ cho hệ thống nhúng.

    Nhược điểm: Phụ thuộc vào môi trường (Dependency). Nếu trên hệ điều hành (mục lục /usr/lib/) không chứa sẵn file .so tương ứng, chương trình sẽ crash ngay lập tức với lỗi error while loading shared libraries.

3. Tích hợp Sysroot (Staging)

Để Cross-Compiler (Trình biên dịch chéo) nhận diện được thư viện tự viết giống như một thư viện chuẩn của hệ thống, ta bắt buộc phải đưa file Header (.h) vào output/staging/usr/include/ và các file thư viện (.a, .so) vào output/staging/usr/lib/. Sysroot chính là môi trường giả lập thư mục gốc của board nhúng nằm trên máy tính Host.
II. Kịch bản Demo thực tế (Dành cho báo cáo/bảo vệ)

Bước 1:Trình bày mã nguồn thư viện và ứng dụng

    Trình bày: "Dạ thưa thầy, em đã tự viết một thư viện toán học đơn giản gồm file mymath.h khai báo hàm 
    và mymath.c định nghĩa hàm cộng hai số. Sau đó, em viết file app_mymath.c để gọi hàm này."

    Lệnh show code: cat mymath.h mymath.c app_mymath.c

Bước 2: Biên dịch và tạo Thư viện Tĩnh & Động

    Trình bày: "Để tạo thư viện tĩnh, em biên dịch file .c ra mã đối tượng (.o) rồi dùng công cụ ar để đóng gói lại thành file .a."
    Bash

    ../output/host/bin/arm-buildroot-linux-gnueabihf-gcc -c mymath.c -o mymath.o
    ../output/host/bin/arm-buildroot-linux-gnueabihf-ar rcs libmymath.a mymath.o

    Trình bày: "Với thư viện động, em phải thêm cờ -fPIC (Position Independent Code) để mã máy có thể linh hoạt địa chỉ trên RAM,
    sau đó liên kết thành file .so bằng cờ -shared."
    Bash

    ../output/host/bin/arm-buildroot-linux-gnueabihf-gcc -fPIC -c mymath.c -o mymath_dynamic.o
    ../output/host/bin/arm-buildroot-linux-gnueabihf-gcc -shared -o libmymath.so mymath_dynamic.o

Bước 3: Tích hợp vào Sysroot và Biên dịch ứng dụng

    Trình bày: "Em đưa file .h và các file thư viện vào thư mục staging của Buildroot. Sau đó, em biên dịch ra 2 phiên bản ứng dụng:
    tĩnh và động."
    Bash

    # Biên dịch bản Động (Linker tự ưu tiên tìm file .so)
    ../output/host/bin/arm-buildroot-linux-gnueabihf-gcc app_mymath.c -o app_dynamic -lmymath

    # Biên dịch bản Tĩnh (Ép Linker dùng file .a)
    ../output/host/bin/arm-buildroot-linux-gnueabihf-gcc app_mymath.c ../output/staging/usr/lib/libmymath.a -o app_static

Bước 4: Chứng minh sự khác biệt về Dependency (Phụ thuộc)

    Trình bày: "Sử dụng công cụ readelf, ta có thể thấy rõ bản app_dynamic yêu cầu hệ điều hành phải có sẵn libmymath.so lúc khởi chạy,
    trong khi bản app_static thì không hề có yêu cầu này vì nó đã nhúng mã nguồn vào trong."

    Lệnh kiểm tra:
    Bash

    readelf -d app_dynamic | grep mymath  # Sẽ hiện Shared library: [libmymath.so]
    readelf -d app_static | grep mymath   # Không trả về kết quả nào

Bước 5: Triển khai xuống board BeagleBone Black và Chạy thử

    Dựng Local Web Server trên máy Host: python3 -m http.server 8080

    Tải xuống board BBB qua Minicom:
    Bash

    wget http://<IP_HOST>:8080/app_static
    wget http://<IP_HOST>:8080/app_dynamic
    chmod +x app_static app_dynamic

    Trình bày: "Khi chạy app_static, chương trình in ra kết quả ngay lập tức. Nhưng khi chạy app_dynamic,
    hệ thống báo lỗi thiếu thư viện (not found). Để khắc phục, em kéo file libmymath.so thả vào đúng thư mục /usr/lib/ của board."
    Bash

    ./app_static      # Chạy thành công
    ./app_dynamic     # Báo lỗi thiếu thư viện
    cd /usr/lib
    wget http://<IP_HOST>:8080/libmymath.so
    cd /root
    ./app_dynamic     # Chạy thành công
