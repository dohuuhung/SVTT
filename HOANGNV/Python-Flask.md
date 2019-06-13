# Flask

## 1. Cài đặt Flask
Tạo project folder và folder venv bên trong:
```
mkdir myproject
cd myproject
python3 -m venv venv
```
Khởi động virtualenv
```
. venv/bin/activate
```
Cài đặt Flask
```
pip install Flask
```
Chạy ứng dụng
```
$ export FLASK_APP=hello.py
$ flask run
 * Running on http://127.0.0.1:5000/
```
Public server:
```
flask run --host=0.0.0.0
```

## 2. Routing
- Sử dụng hàm route() để chỉ định url tới 1 function
```python3
@app.route('/')
def index():
    return 'Index Page'

@app.route('/hello')
def hello():
    return 'Hello, World'
```

- Thêm tham số vào URL bằng cách sử dụng cấu trúc <variable_name> hoặc thêm kiểu cho tham số bằng <converter:variable_name>
```python3
@app.route('/user/<username>')
def show_user_profile(username):
    # show the user profile for that user
    return 'User %s' % username

@app.route('/post/<int:post_id>')
def show_post(post_id):
    # show the post with the given id, the id is an integer
    return 'Post %d' % post_id

@app.route('/path/<path:subpath>')
def show_subpath(subpath):
    # show the subpath after /path/
    return 'Subpath %s' % subpath
```

## 3. Phương thức HTTP 
Sử dụng tham số methods để xử lý nhiều phương thức HTTP ngoài GET
```python3
@app.route('/login', methods=['GET', 'POST'])
def login():
    if request.method == 'POST':
        return do_the_login()
    else:
        return show_the_login_form()
```

## 4. Static Files
Các file CSS và Javascript được lưu trong folder static, được khởi tạo URL bằng:
```python3
url_for('static', filename='style.css')
```

## 5. Rendering Templates
- Để hiển thị nội dung file html, sử dụng hàm render_template()
```python3
from flask import render_template
 
@app.route('/hello/')
@app.route('/hello/<name>')
def hello(name=None):
    return render_template('hello.html', name=name)
```
- File hello.html nằm trong folder templates

## 6. Request Object
- Để lấy dữ liệu form, sử dụng module request
```python3
from flask import request
 
username = request.form['username']
password = request.form['password']
```

- Upload file:
```python3 
from flask import request

@app.route('/upload', methods=['GET', 'POST'])
def upload_file():
    if request.method == 'POST':
        f = request.files['the_file']
        f.save('/var/www/uploads/uploaded_file.txt')
    ...
```

