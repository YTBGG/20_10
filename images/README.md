# Ảnh

Đặt ảnh vào thư mục này, sau đó cập nhật đường dẫn tương đối trong `index.html`:

```js
avatarUrl: "./images/ebe.jpg",
```

Ảnh riêng cho từng kỷ niệm nằm trong thuộc tính `image` của mỗi phần tử trong
`CONFIG.tracks`:

```js
image: "./images/ky-niem-01.jpg"
```

Dùng tên tệp viết thường, không dấu và không có khoảng trắng để tránh lỗi phân
biệt chữ hoa/chữ thường trên GitHub Pages. Ưu tiên ảnh JPG hoặc WebP đã nén để
trang tải nhanh. Tệp `image-placeholder.svg` sẽ được hiển thị khi chưa có ảnh
hoặc ảnh được cấu hình không tải được.
