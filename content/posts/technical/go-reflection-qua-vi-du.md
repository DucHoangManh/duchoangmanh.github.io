---
title: "Go: Reflection qua ví dụ cụ thể"
date: 2022-07-13T00:01:23+07:00
draft: true
---

## Reflection trong Go  

Reflection trong go giúp cho ta có thể theo dõi code tại thời điểm `runtime`, cho phép tiếp cận mã nguồn chương trình dưới dạng data có thể xử lý thay vì các lệnh có thể thực thi (một nhánh trong `metaprogramming`).  
Reflection trong go có thể được thực hiện thông qua `reflect` package.
Một số khả năng của reflect:
- Kiểm tra thông tin của một struct (số lượng method, số lượng field, đọc struct tag...) mà không cần biết trước về struct đó.
- Kiểm tra và cập nhật một type (slice, channel, struct, interface, pointer...) mà không cần biết trước về type đó.   

Đổi lại những khả năng mạnh mẽ của recflect, performance và tính maintainable của code sẽ giảm, cần cân nhắc kĩ trước khi quyết định đưa vào ứng dụng.
> Clear is better than clever. Reflection is never clear - Go proverb (Rob Pike) 

Ứng dụng của reflect:
- Các function, method với đầu vào không rõ trước.
- Viết công cụ phân tích mã nguồn.
- Thực thi code linh hoạt (VD liệt kê các method của một struct và gọi method theo tên).

Một số package, library sử dụng reflection: `fmt`, `encoding/json`, `gorm`, `sqlx`...