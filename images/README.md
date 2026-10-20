# Album ảnh riêng tư

Ảnh cá nhân không được lưu trong repository/GitHub Pages. Ứng dụng chỉ tải ảnh
từ bucket Supabase riêng sau khi đăng nhập, dùng URL ký có thời hạn 5 phút.
Khách chưa đăng nhập không được đọc bảng ảnh hoặc xem ảnh trong Storage.

## Thiết lập Supabase

1. Trong **Authentication settings**, tắt đăng ký công khai. Trong **Authentication
   → Users**, chỉ giữ hoặc tạo tài khoản cho những người được phép xem album; xóa
   tài khoản lạ nếu trước đây đã mở đăng ký. Mỗi người nên có mật khẩu riêng.
2. Mở **SQL Editor**, chạy toàn bộ SQL ở cuối hướng dẫn **trước khi triển khai
   phiên bản trang này**. Script chuyển bucket `site-photos` sang private và thay
   quyền đọc công khai bằng quyền yêu cầu phiên đăng nhập. Có thể chạy lại nếu cần
   áp dụng chính sách mới.
3. Trong **Authentication → URL Configuration**, cho phép URL website GitHub
   Pages làm site URL/redirect URL nếu Supabase Authentication yêu cầu.
4. Đăng nhập vào trang được triển khai. Ảnh được tải lên từ thư viện sẽ nằm trong
   bucket private; chỉ người có tài khoản được tạo trong dự án mới xem được.

## Đặt lại mật khẩu

1. Trong Supabase, mở **Authentication → URL Configuration**. Đặt **Site URL** là
   `https://ytbgg.github.io/20_10/` và thêm
   `https://ytbgg.github.io/20_10/**` vào **Redirect URLs**. Chỉ thêm
   `http://localhost:3000/**` nếu cần thử trên máy local.
2. Giữ mẫu email **Reset Password** mặc định dùng `{{ .ConfirmationURL }}` để
   Supabase xác minh liên kết trước khi chuyển về trang.
3. Trên trang, nhập email tài khoản rồi bấm **Quên mật khẩu?**. Mở email mới nhất,
   bấm liên kết và đặt mật khẩu mới trên trang album. Liên kết đặt lại cũ hoặc đã
   hết hạn không dùng được; hãy yêu cầu email mới.

`index.html` dùng Supabase Project URL và publishable key ở trình duyệt. Publishable
key không phải mật khẩu; RLS và bucket private mới là lớp bảo vệ. Không bao giờ
đưa `service_role` key, secret key hoặc mật khẩu vào source code/Git.

## SQL tạo bảng và chuyển ảnh sang private

```sql
create table if not exists public.photo_library (
  id uuid primary key default gen_random_uuid(),
  object_path text not null unique,
  file_name text not null,
  uploaded_by uuid not null references auth.users(id),
  created_at timestamptz not null default now()
);

create table if not exists public.site_photo_settings (
  id boolean primary key default true check (id = true),
  avatar_photo_id uuid references public.photo_library(id) on delete set null,
  cover_photo_id uuid references public.photo_library(id) on delete set null,
  track_photo_ids jsonb not null default '{}'::jsonb
    check (jsonb_typeof(track_photo_ids) = 'object')
);

alter table public.site_photo_settings
  add column if not exists cover_photo_id uuid
  references public.photo_library(id) on delete set null;

insert into public.site_photo_settings (id)
values (true)
on conflict (id) do nothing;

alter table public.photo_library enable row level security;
alter table public.site_photo_settings enable row level security;

revoke all on public.photo_library, public.site_photo_settings from anon, public;
grant select, insert, delete on public.photo_library to authenticated;
grant select, insert, update on public.site_photo_settings to authenticated;

drop policy if exists "Public can view photo library" on public.photo_library;
drop policy if exists "Authenticated admins can view photo library" on public.photo_library;
drop policy if exists "Authenticated admins can add photo records" on public.photo_library;
drop policy if exists "Authenticated admins can delete photo records" on public.photo_library;
drop policy if exists "Authenticated users can view photo library" on public.photo_library;
drop policy if exists "Authenticated users can add photo records" on public.photo_library;
drop policy if exists "Authenticated users can delete photo records" on public.photo_library;
create policy "Authenticated users can view photo library"
on public.photo_library for select
to authenticated
using (auth.uid() is not null);

create policy "Authenticated users can add photo records"
on public.photo_library for insert
to authenticated
with check (uploaded_by = auth.uid());

create policy "Authenticated users can delete photo records"
on public.photo_library for delete
to authenticated
using (auth.uid() is not null);

drop policy if exists "Public can view photo assignments" on public.site_photo_settings;
drop policy if exists "Authenticated admins can view photo assignments" on public.site_photo_settings;
drop policy if exists "Authenticated admins can initialize photo assignments" on public.site_photo_settings;
drop policy if exists "Authenticated admins can update photo assignments" on public.site_photo_settings;
drop policy if exists "Authenticated users can view photo assignments" on public.site_photo_settings;
drop policy if exists "Authenticated users can initialize photo assignments" on public.site_photo_settings;
drop policy if exists "Authenticated users can update photo assignments" on public.site_photo_settings;
create policy "Authenticated users can view photo assignments"
on public.site_photo_settings for select
to authenticated
using (auth.uid() is not null);

create policy "Authenticated users can initialize photo assignments"
on public.site_photo_settings for insert
to authenticated
with check (id = true);

create policy "Authenticated users can update photo assignments"
on public.site_photo_settings for update
to authenticated
using (auth.uid() is not null)
with check (id = true);

insert into storage.buckets (id, name, public, file_size_limit, allowed_mime_types)
values (
  'site-photos',
  'site-photos',
  false,
  15728640,
  array['image/jpeg', 'image/png', 'image/webp', 'image/gif']
)
on conflict (id) do update set
  public = false,
  file_size_limit = excluded.file_size_limit,
  allowed_mime_types = excluded.allowed_mime_types;

revoke all on storage.objects from anon, public;
grant select, insert, delete on storage.objects to authenticated;

drop policy if exists "Public can view site photos" on storage.objects;
drop policy if exists "Authenticated admins can view site photos" on storage.objects;
drop policy if exists "Authenticated admins can upload their site photos" on storage.objects;
drop policy if exists "Owners can remove their site photos" on storage.objects;
drop policy if exists "Authenticated admins can delete site photos" on storage.objects;
drop policy if exists "Authenticated users can view site photos" on storage.objects;
drop policy if exists "Authenticated users can upload site photos" on storage.objects;
drop policy if exists "Authenticated users can delete site photos" on storage.objects;

create policy "Authenticated users can view site photos"
on storage.objects for select
to authenticated
using (bucket_id = 'site-photos' and auth.uid() is not null);

create policy "Authenticated users can upload site photos"
on storage.objects for insert
to authenticated
with check (
  bucket_id = 'site-photos'
  and (storage.foldername(name))[1] = auth.uid()::text
);

create policy "Authenticated users can delete site photos"
on storage.objects for delete
to authenticated
using (bucket_id = 'site-photos' and auth.uid() is not null);
```

The script revokes anonymous table access, sets the storage bucket private, and
allows authenticated project users to view, upload, and delete images. Disable
public sign-up in Supabase Auth and create only accounts for trusted viewers.

## Giới hạn cần biết

- GitHub Pages serves the HTML/JavaScript source publicly. The login gate hides
  the app interface; it cannot make the source code itself private.
- `robots.txt` and the page's `noindex` metadata ask compliant crawlers not to
  fetch, index, or archive it. This is only a crawler hint, not access control.
- Files committed to the repository in the past may remain available through
  Git history, forks, caches, or existing copies after removal from the latest
  commit. If the repository was public, remove the old image blobs from Git
  history and rotate/recreate the public repository if necessary.
- Signed image links expire after five minutes, but someone who is allowed to
  view an image can still save it or take a screenshot while signed in.
- `image-placeholder.svg` is a generic placeholder, not a personal photo, and
  can remain in the public repository.
