---
title: "System Design for Blog with Massive Users"
date: 2022-05-16T01:39:17+07:00
draft: true
author:
tags:
    - "blog"
    - "story"
    - "research"
description:
toc:
---
Một bài tập và là một vấn đề vô cùng thú vị trong những ngày đầu của hành trình Fresher tại ZaloPay - VNG. <!--more-->

## Problem statement

Câu chuyện về ZaloPay hay VNG mình xin phép được kể sau ở những bài viết khác. Hiện tại, mình sẽ chỉ tập trung vào phần giải quyết vấn đề đã đặt ra ở tiêu đề hơn là kể chuyện, và nó cũng chính là một trong những bài tập của tuần training đầu tiên tại ZaloPay. 

> Imagine you have to design a system for serving blogs to massive readers (10k tps). How would you design the system?

## Analysis

10k *tps*? *tps* ở đây là gì nhỉ? 10k users à?

Sau một hồi tìm hiểu thì mình biết rằng *tps* có các ý nghĩa khác nhau trong các lĩnh vực khác nhau và ở đây ta hiểu *tps* chính là từ viết tắt của *Transaction Per Second*. 

Vì vậy, vấn đề ở đây đặt ra rằng system của chúng ta phải handle được tầm 10k truy vấn/transaction đồng thời, tức là phục vụ cho khoảng 10,000 người như các bạn ở đây cùng đọc tại một thời điểm. 

Ta sẽ cùng nhau trả lời một số câu hỏi để có cái nhìn tổng quan hơn về hệ thống mà ta đang muốn thiết kế.

1. *Who is going to use it?* Các bạn, các đọc giả thân yêu của tôi :blush:
2. *How are they going to use it?* Truy cập, lướt, đọc, điều hướng, lặp lại.
3. *How many users are there?* Sure là phải lớn hơn nhiều số lượng 10k tps, cứ cho là 1M - 10M users nhé :kissing:
4. *What does the system do?* Gửi/nhận các request từ người dùng, lưu trữ/cập nhật dữ liệu bài viết, xử lí lượng lớn truy vấn, v.v.
5. *What are the inputs and outputs of the system?* Đầu vào là truy vấn từ người dùng, đầu ra là nội dung của trang web, bài viết tương ứng với truy vấn.
6.  *How much data do we expect to handle?* Blog cá nhân, mình viết đến năm 100 tuổi thì ước chừng 1TB chắc là đủ (bao gồm cả text, script, hình ảnh, ...)
7.  *How many requests per second do we expect?* 10,000 tps
8.  *What is the expected read to write ratio?* 10000:1 

Kì vọng cho system của chúng ta, mình sẽ tóm gọn như sau:
- **Scalability**: có khả năng đáp ứng và xử lí lượng lớn user với tốc độ ổn định.
- **Availability**: luôn có thứ để hiển thị cho user, có thể không phải là nội dung mới nhất nhưng vẫn đảm bảo việc luôn hiển thị cho user.
- **Partition Tolerance**: hệ thống vẫn tiếp tục hoạt động trong trường hợp có lỗi xảy ra.

## Solution

Chúng ta sẽ tiếp cận ở góc độ high-level về hệ thống mà chúng ta đang nhắm tới, phân rã nó theo từng scenario và mức growth nhất định.

### > Basic system (10 to 100 users)

Hãy bắt đầu với một thiết kế cơ bản nhất của hệ thống chúng ta, một blog dành cho 10 đến 100 users. 

![Basic system](/images/blogs/001/basic.png) 

Công việc lúc này là vô cùng đơn giản khi ta có thể setup một self-host server hoặc ở đâu đó trên cloud, và khi user cần truy cập vào trang web của chúng ta, DNS lúc này sẽ vào việc và trả về địa chỉ IP của server, qua đó trả về dữ liệu tương ứng cho user (HTML, CSS, Script, nội dung blog, v.v.). 

Như chúng ta thấy, dữ liệu của system được lưu trữ tại server, có thể là các folder chứa các tệp text, hình ảnh, v.v. Hoặc là một NoSQL database được cấu hình sẵn (*sử dụng NoSQL vì dữ liệu của blog có tính semi-structured và tổng dung lượng tăng trưởng dự kiến khoảng 1TB*), điều này tiềm ẩn một số rủi ro nhất định, tuy nhiên giải pháp vẫn có thể chấp nhận được với ít chi phí + nỗ lực nhất, và performance ở đây cũng tương tự, có thể chấp nhận được.

### > Scale up to 10,000 users & 100 to 500 tps!

Lúc này, blog chúng ta đã lớn mạnh, lượng user đã đạt tới con số 10,000 và throughput (thông lượng) cần đáp ứng khoảng 100-500 tps. 

Và đây chính là lúc mà chúng ta thật sự cần quan tâm đến scalability, scale cái gì, scale như thế nào, để đạt được hiệu quả.

#### Server Scaling

Ban đầu, ta chỉ sử dụng một server duy nhất vì một lí do đơn giản là lượng user chưa nhiều -> số lượng truy vấn phát sinh không nhiều. 

Nhưng hiện tại, lượng user đã tăng trưởng lên gấp 10 -> số lượng truy vấn tăng -> nếu dùng single server thì sẽ tăng khối lượng công việc cho worker (server) -> tăng latency -> giảm performance. Tệ hơn là server có thể bị overload, dẫn đến hỏng hóc -> nguy cơ mất sạch dữ liệu tăng cao.

Giải pháp ở đây là phải tăng khả năng giải quyết và xử lí công việc của worker, để thuận lợi hơn ta sẽ sử dụng các cloud service để giảm thiểu chi phí bảo trì, bảo dưỡng và vận hành. Tiếp theo đó ta có 2 options để lựa chọn cho việc Server Scaling:
- *Vertical Scaling*: upgrade phần cứng, thêm RAM, RAM và RAM, sử dụng CPU tốc độ cao hơn, etc.
- *Horizontal Scaling*: ta sử dụng thêm các máy server, kết nối chúng lại với nhau.

![Scale up level 1](/images/blogs/001/intermediate.png)

Ở đây, mình sẽ lựa chọn giải pháp ở sau, *Horizontal Scaling*, vì một số lí do chính sau:
- Thường thì sẽ có lợi hơn về mặt chi phí.
- Tạo tiền đề cho việc scale khi tiếp tục tăng trưởng và có nhu cầu.
- Tăng khả năng chịu lỗi của system, hệ thống vẫn có thể hoạt động tiếp tục nếu có 1 vài sự cố xảy ra với một trong các server -> giảm downtime.

Bên cạnh đó, ta sẽ sử dụng *Reversed Proxy* layer ở giữa Client và Server để tăng độ bảo mật, tăng scalability và flexibility của system cũng như đóng vai trò như một *Load Balancer*. Ở đây, mình không dùng *Load Balancer* vì mức user growth chưa đạt đến mức cần phải scale out quá nhiều, do đó ta chỉ cần *Reversed Proxy* là đủ.


#### Database Scaling

Ở góc độ Database, liệu chúng ta có cần phải scailing như cách chúng ta đã làm với Server? 

Well, để trả lời câu hỏi này, chúng ta hãy nhìn lại các câu hỏi đã được nêu ra ở trên. Vì đặc điểm của application (blog), việc tăng trưởng user sẽ không làm tăng số lượng dữ liệu phát sinh tương ứng, mà chỉ làm tăng số lượng transaction giữa client và server, tương đương với việc tăng số lượng truy vấn của server lên database, ngoài ra khối lượng dữ liệu dự đoán là không quá lớn (1TB). 

Thế nên, ở mức tăng trưởng user này, ta chỉ cần optimize truy vấn và tăng tốc độ truy vấn của server với database. Ở đây mình sẽ sử dụng thêm Cache Layer và cấu hình lại logic truy vấn của server.

### > Our goal: 10,000 tps! Massive users: 1M to 10M in expectation

Mức growth này là thử thách tiếp theo mà chúng ta phải đương đầu, vậy vấn đề lớn nhất ở đây là gì? 10,000 tps hay là 10M users? Theo lượng kiến thức hiện tại và đánh giá chủ quan mà mình đang có, thì để handle 10,000 tps, ta có thể scale out hệ thống hiện tại thêm một chút, có thể là từ 3 tăng lên 10 hay 20 server. 

Vậy vấn đề thật sự nằm ở việc 10M users ư? Maybe là không, như đã phân tích trước đó, ta không lưu trữ bất cứ thông tin nào về user, do đó nó sẽ không ảnh hưởng đến kích thước của database khi lượng user tăng lên. 

Điều mà mình nghĩ là vấn đề ở đây khi đạt mức growth này chính là availability, eventual consistency. Tại sao lại như vậy? Lượng users tăng cao, đồng nghĩa với việc lượng traffic tăng cao, ta không thể đảm bảo tất cả người dùng đều có thể truy cập vào trang blog của chúng ta với performance ổn định vì một lí do đơn giản: **network is not reliable**, do đó ta phải thiết kế và scale hệ thống để giải quyết các vấn đề vừa nêu.

Do đó, ta sẽ sử dụng Load Balancer thay cho Reversed Proxy ở giữa Client và Server vì số lượng server đã tăng lên đáng kể, đảm bảo việc điều hướng và thực hiện các công việc được diễn ra trơn tru trên các worker, tăng khả năng chịu lỗi của toàn bộ system.

Ngoài ra, ta sẽ thực hiện **Sharding** cũng như **SQL Tuning** database của chúng ta, điều này sẽ giúp cho tốc độ truy vấn tăng lên đáng kể, tăng availability của system. Ta cũng có thể thêm vào đây một vài dạng **replication** cho database để tránh việc mất mát dữ liệu, tăng độ hài lòng và trải nghiệm người dùng.

![Advanced System](/images/blogs/001/advanced.png)

## Conclusion
Như vậy, ta đã giải quyết vấn đề đặt ra ở tiêu đề blog từng bước một, từ một hệ thống basic nhất, scale đến level phức tạp hơn.
Bản thân mình luôn biết rằng, không có thứ gì là hoàn hảo, do đó System Design cũng vậy, ta phải chấp nhân trade-off trên một số phương diện khác nhau để đạt được mục đích cao hơn của hệ thống.

Ngoài ra, mình cũng muốn nhấn mạnh rằng, những nhận định và chia sẻ ở trên hoàn toàn là từ quá trình mình tự research và tư duy chủ quan của bản thân mình, có thể sẽ có những chỗ không hợp lí, không tối ưu hay thậm chí là sai bản chất. Vì vậy, mong bạn đọc hãy luôn kiểm chứng và đối chiếu với kiến thức mà mình đã có, cũng như những kiến thức tương tự ở ngoài kia để đảm bảo rằng chúng ta sẽ có được nhiều thông tin và lượng kiến thức bổ ích nhất có thể.

Lời kết, mình xin cảm ơn các bạn vì đã dành thời gian đọc bài viết này, nó là một bài tập cũng như là một kỉ niệm đáng giá trong career path của mình, mình tin là vậy. 

Wish all the best for you! Hẹn gặp lại!

## References
- [System Design Primer](https://github.com/donnemartin/system-design-primer)
- [Scailability for Dumies](https://www.lecloud.net/tagged/scalability/chrono)
- [System Design Interviews](https://www.youtube.com/watch?v=YkGHxOg9d3M)