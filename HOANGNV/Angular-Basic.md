# AngularJS

## 1. Giới thiệu
Angular là 1 platform và framework dùng để xây dựng ứng dụng phía client bằng HTML và TypeScript. Angular được viết bằng ngôn ngữ TypeScript.

## 2. Cài đặt
B1. Cài Angular CLI
`npm install -g @angular/cli
`
B2. Tạo workspace và ứng dụng
`ng new my-app
`
B3. Chạy ứng dụng
`cd my-app
ng serve --open
`

## 3. Khái niệm
### 3.1. Module
- Các thành phần cơ bản trong Angular bao gồm các NgModule. Mỗi ứng dụng có ít nhất 1 root module (AppModule) dùng để khởi động, và nhiều module chức năng.
- Module chứa các component, service provider và các file code khác. Các module có thể sử dụng các function của nhau.
- NgModule Metadata: Mỗi NgModule được định nghĩa bằng các thuộc tính:
    - declarations: các component, directive và pipe 
    - exports: Các khai báo có thể được sử dụng ở các module khác
    - imports: Các module khác được sử dụng
    - providers: Các service được sử dụng toàn cục
    - bootstrap: Root component, chỉ nên khai báo trong root NgModule

- Định nghĩa 1 root NgModule:
```javascript
import { NgModule }      from '@angular/core';
import { BrowserModule } from '@angular/platform-browser';
@NgModule({
  imports:      [ BrowserModule ],
  providers:    [ Logger ],
  declarations: [ AppComponent ],
  exports:      [ AppComponent ],
  bootstrap:    [ AppComponent ]
})
export class AppModule { }
```

### 3.2. Component
- Các component là các class chứa logic và dữ liệu, liên kết với HTML template dùng để định nghĩa 1 view (thành phần giao diện) 
- Component metadata:
	- selector: Tên CSS selector trong HTML template
	- templateUrl: đường dẫn file HTML template
	- providers: các services sử dụng trong component

```javascript
@Component({
  selector:    'app-hero-list',
  templateUrl: './hero-list.component.html',
  providers:  [ HeroService ]
})
export class HeroListComponent implements OnInit {
/* . . . */
}
```
	
### 3.3. Template và view 
- Template dùng để định nghĩa cấu trúc HTML của mỗi component
- Template cung cấp 2 kiểu truyền dữ liệu (data binding):
	- Truyền sự kiện (Event binding): phản hồi các input của user từ phía view
	- Truyền thuộc tính (Property binding): truyền giá trị từ dữ liệu vào HTML
	
- Cú pháp của template:
	- *ngIf, *ngFor, *ngSwitch, *ngClass, *ngStyle
	- {{ }}: interpolation
	- (event): event binding
	- [property]: property binding
	- [(ng-model)]: truyền dữ liệu 2 chiều giữa input và component
	
```
<ul>
  <li *ngFor="let hero of heroes" (click)="selectHero(hero)">
    {{hero.name}}
  </li>
</ul>
<app-hero-detail *ngIf="selectedHero" [hero]="selectedHero"></app-hero-detail>
<li (click)="selectHero(hero)"></li>
<input [(ngModel)]="hero.name">
```
	
### 3.4. Pipe
- Pipe dùng để chuyển đổi dữ liệu khi hiển thị trên template, VD như ngày tháng, đơn vị tiền tệ,...

```
<!-- fullDate format: output 'Monday, June 15, 2015'-->
<p>The date is {{today | date:'fullDate'}}</p>
```

### 3.5. Service
- Service class không liên kết với 1 view cụ thể mà được sử dụng trong tất cả các component
- Nhiệm vụ của service class: lấy dữ liệu từ server, xác thực user input, in log ra console

Định nghĩa 1 service class:
```javascript
import { Injectable } from '@angular/core';
 
@Injectable()
export class TimeService {
  constructor() { }
 
  getTime(){
    return `${new Date().getHours()} : ${new Date().getMinutes()} : ${new Date().getSeconds()}`;
  }
}
```

### 3.6. Routing
- Trong 1 route template, các component được gắn với 1 URL. Khi chuyển đến 1 URL, component sẽ được load mà không cần tải lại trang.
- Các thuộc tính của Route:
	+ path: đường dẫn URL
	+ component: component ứng với URL trên
	+ redirectTo: chuyển hướng đến 1 URL khác 
- Thêm vị trí của component cần render trong HTML template:
```
<router-outlet></router-outlet>
```
