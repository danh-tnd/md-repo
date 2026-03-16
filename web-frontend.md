
Module `modules/leia-web-frontend` là phần frontend WebUI của hệ thống Leia, đóng vai trò build và đóng gói tài nguyên web (JS, CSS, HTML, resources tĩnh) thành artifact Java để deploy cùng backend.

Dưới đây là cách nó được tổ chức và công nghệ liên quan.

---

## 1. Mục đích và vai trò

- Cung cấp toàn bộ **UI tĩnh** (HTML template, JavaScript, CSS, images, v.v.) cho ứng dụng Leia.
- Build frontend bằng **Node.js + Gulp** rồi đóng gói vào một **JAR** (`leia-web-frontend`) để các module web backend có thể sử dụng (thường copy vào `META-INF/resources` để servlet container serve).

---

## 2. Công nghệ chính

### 2.1. Maven + frontend-maven-plugin

- `modules/leia-web-frontend/pom.xml`:
  - Là một Maven module kiểu `jar`.
  - Sử dụng **frontend-maven-plugin** (`com.github.eirslett:frontend-maven-plugin`) để:
    - Cài **Node.js** version cấu hình (hiện là `v18.20.4`).
    - Cài **npm** tương ứng (`9.8.1`).
    - Chạy các lệnh npm/gulp trong pipeline Maven (ví dụ `npm install`, `npx gulp build`).
- Maven chịu trách nhiệm:
  - Quản lý lifecycle Java (compile, test, package) của module.
  - Gọi pipeline Node/Gulp để build frontend và đặt output vào `target`.

### 2.2. Node.js + NPM

- Module có `package.json`:
  - DevDependencies chứa các package build:
    - `gulp`, `gulp-cli`, `gulp-babel`, `gulp-uglify`, `gulp-clean-css`, `gulp-htmlmin`, `gulp-rev`, `gulp-sourcemaps`, `merge-stream`, `cloneable-readable`, v.v.
    - Các tool khác: `eslint`, `gulp-jshint`, `gulp-jasmine`, `gulp-istanbul`, `requirejs`, `lazypipe`, v.v.
  - Một số package cũ (như `gulp-minify-css`, `gulp-minify-html`) là di sản; code đã chuyển sang clean-css + htmlmin.

### 2.3. Gulp 4

- `modules/leia-web-frontend/gulpfile.js` dùng Gulp 4:
  - Định nghĩa các task:
    - `clean` – xóa `./target/build`.
    - `messagefilebuild` – build file message JS từ spec.
    - `csslint`, `cssbuild` – lint và build CSS.
    - `jshint`, `jasmine`, `jsbuild`, `coverage` – lint, test, build và coverage cho JS.
    - `htmlbuild` – build HTML từ templates, inject manifest.
    - `resourcebuild` – copy resources tĩnh (lib JS/CSS, txt, html) vào `target/build`.
    - `package` – đóng gói tất cả vào `./target/app/META-INF/resources` + file `files.list`.
    - `build` – chuỗi tổng hợp: `clean → jsbuild → cssbuild → htmlbuild → resourcebuild → package`.
    - `default` – alias của `build`.
  - Sử dụng `gulp.series` để đảm bảo thứ tự và xử lý async đúng chuẩn Gulp 4 (mỗi task phải return stream/promise).

---

## 3. Cấu trúc thư mục chính

Trong `modules/leia-web-frontend`:

- `buildlib/` – nơi chứa **các strategy build** (tập trung logic Gulp theo pattern Strategy):
  - `JSStrategy.js` – build JS.
  - `CSSStrategy.js` – build CSS.
  - `HTMLStrategy.js` – build HTML.
  - `MessageFileStrategy.js` – generate JS messages từ spec i18n.
  - `ResourceStrategy.js` – copy resources tĩnh.
  - `PackageStrategy.js` – đóng gói output vào cấu trúc deploy.
  - Utility:
    - `JSONUtil.js` – đọc JSON/manifest an toàn (tránh crash khi file thiếu/hỏng).
    - `ObjectUtil.js`, `MessageFormatter.js`, `Params.js` – helper.
- `sources.js` (hoặc `sources/index.js`) – định nghĩa các path nguồn:
  - Ví dụ: `paths.js`, `paths.css`, `paths.templates`, `paths.strings`, `templateRoot`, v.v.
- `buildconfigs.js` – cấu hình build (ví dụ, mapping config cho templates).
- `lib/` – chứa JS/CSS library gốc (nguồn để build/gom và copy).
- `templates/` – HTML template (có thể dùng ECT hoặc template engine khác).
- `strings/` hoặc tương đương – file spec message đa ngôn ngữ.
- `images/` – ảnh tĩnh.

Sau khi chạy `gulp build`, output chính:
- `target/build/`
  - `scripts.js`, `scripts.js.gz`, `scripts.js.manifest`
  - `styles.css`, `styles.css.gz`, `styles.css.manifest`
  - `messages*.js`, `messages.{language}.manifest`
  - HTML đã minify + gzip.
  - Các file resource tĩnh (JS/CSS/HTML từ `lib/`).
- `target/app/META-INF/resources/`
  - Cấu trúc tài nguyên để deploy kèm WAR/backend.
  - `files.list` – danh sách file cho packaging.

---

## 4. Chi tiết các Strategy chính

### 4.1. JSStrategy

`buildlib/JSStrategy.js`:

- `jshintTask(srcs)`:
  - Lint JS bằng `gulp-jshint` với `lookup:false`.
- `jasmineTask(specs)`:
  - Chạy unit test JavaScript bằng `gulp-jasmine`.
- `coverageTask(srcs, specs, opts)`:
  - Dùng `gulp-istanbul` để instrument source.
  - Chạy Jasmine test trên code đã instrument.
  - Xuất report coverage.
- `manifestName(outFile)`:
  - Tạo tên manifest cho file JS, ví dụ `scripts.js` → `scripts.js.manifest`.
- `jsTask(srcs, opts)`:
  - Tạo pipeline:
    1. `gulp.src(srcs)`
    2. (optional) `sourcemaps.init()` nếu `opts.debug`.
    3. `concat(outFile)` → `uglify()` → `rev()`.
    4. (optional) `sourcemaps.write()`.
    5. Bọc stream bằng `cloneable-readable`.
  - Từ stream base đó, tạo 3 nhánh:
    - `.js` – ghi file JS bình thường.
    - `.gz` – gzip rồi ghi.
    - `.manifest` – tạo manifest JSON mapping file gốc → file rev’ed (hash).
  - Dùng `merge-stream` merge 3 nhánh và return cho Gulp 4.

### 4.2. CSSStrategy

`buildlib/CSSStrategy.js`:

- `csslintTask(srcs)`:
  - Lint CSS bằng `gulp-csslint`.
- `manifestName(outFile)`:
  - `styles.css` → `styles.css.manifest`.
- `cssTask(srcs, opts)`:
  - Tương tự jsTask:
    - Base stream:
      - `gulp.src(srcs)` → `concat(outFile)` → `gulp-clean-css` (minify) → `rev()`.
    - Clone ra 3 nhánh: `.css`, `.gz`, `.manifest`.
    - Merge bằng `merge-stream`.

### 4.3. HTMLStrategy

`buildlib/HTMLStrategy.js`:

- Đọc templates từ `paths.templates`.
- Kết hợp với:
  - Thông tin strings,
  - Manifest JS/CSS,
  - Build configs.
- Dùng `gulp-ect-simple` (hoặc tool tương tự) để render template.
- Minify HTML bằng `gulp-htmlmin` (`collapseWhitespace: true`, v.v.).
- Ghi HTML (và gzip, manifest nếu cần) vào `target/build`.

### 4.4. JSONUtil

`buildlib/JSONUtil.js`:

- `readAsJson(src)`:
  - Nếu file không tồn tại → trả `{}`.
  - Nếu file rỗng → `{}`.
  - Nếu JSON parse lỗi → log warning, trả `{}`.
- Mục đích: tránh build crash khi manifest chưa tồn tại / bị ghi dở.

---

## 5. Gulpfile – Pipeline tổng thể

`gulpfile.js` nối các Strategy lại thành một pipeline rõ ràng:

1. **clean**
   - Xóa `./target/build` để build sạch.

2. **messagefilebuild**
   - Generate `messages.js` và manifest từ file strings spec.
   - Output: `./target/build/messages*.js`, `messages.{language}.manifest`.

3. **csslint**, **cssbuild**
   - Lint & build CSS:
     - `styles.css`, `styles.css.gz`, `styles.css.manifest`.

4. **jshint**, **jasmine**, **jsbuild**
   - Lint JS → chạy test → build:
     - `scripts.js`, `scripts.js.gz`, `scripts.js.manifest`.

5. **coverage**
   - Chạy coverage (optional task).

6. **htmlbuild**
   - Phụ thuộc: `jsbuild`, `cssbuild`, `messagefilebuild`.
   - Đảm bảo:
     - `./target/build` tồn tại.
     - Manifest `scripts.js.manifest`, `styles.css.manifest` tồn tại (tạo `{}` nếu thiếu).
   - Gọi `HTMLStrategy.htmlTask` để render HTML, minify, ghi vào `./target/build`.

7. **resourcebuild**
   - Copy các file resources từ `./lib` (JS/CSS/HTC/TXT/HTML) vào `./target/build`.

8. **package**
   - Phụ thuộc: `htmlbuild`, `resourcebuild`.
   - Gọi `PackageStrategy` để:
     - Copy tất cả JS/CSS/HTML/gz và resources vào `./target/app/META-INF/resources`.
     - Tạo file `files.list`.

9. **build**, **default**
   - `build` = `clean → jsbuild → cssbuild → htmlbuild → resourcebuild → package`.
   - `default` = `build`.

---

## 6. Luồng tích hợp với Maven

- Khi chạy từ root project:
  - `mvn clean install -Duser.timezone=Asia/Tokyo`:
    - Maven vào module `leia-web-frontend`.
    - `frontend-maven-plugin`:
      - Cài Node / npm (theo version khai báo).
      - Chạy `npm install` dựa trên `package.json`.
      - Chạy `npx gulp build` (hoặc lệnh cấu hình).
    - Sau khi Gulp build xong, Maven đóng gói output thành JAR.

- JAR `leia-web-frontend` sau đó được các module WAR/backend sử dụng (copy vào `META-INF/resources` hoặc path tương đương) để serve static content.

---

Nếu bạn muốn, mình có thể vẽ thêm một sơ đồ pipeline (từng bước từ source → target/build → target/app/META-INF/resources) hoặc đi sâu vào một Strategy cụ thể (ví dụ JSStrategy) với từng dòng code.
<!--stackedit_data:
eyJoaXN0b3J5IjpbNTU3NjY0Nzg0XX0=
-->