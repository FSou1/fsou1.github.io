---
title: Angular 8 - Role-based authorization tutorial
tags: Angular8 TypeScript
redirect_from: "/Angular_8_role_based_authorization/"
---

In this post, I'd like to show you an example of how you can implement role-based authorization/access control front end using Angular 8.

## TL;DR

- [Stackblitz - sample](https://stackblitz.com/edit/angular-8-role-based-authorization-sample)
- [GitHub - source code](https://github.com/FSou1/angular-8-role-based-authorization-sample)
- [User role directive](#user-role-directive)
- [Auth guard](#auth-guard)

## Demo in action

![angular-8-rba-authorization](/assets/images/posts/angular%208%20-%20rba%20authorization.gif)

Here is the result:

<iframe width="740" height="530" src="https://stackblitz.com/edit/angular-8-role-based-authorization-sample?embed=1&file=src/app/services/auth.service.ts" frameborder="0" class="center-image"></iframe>

(See on StackBlitz at [https://stackblitz.com/edit/angular-8-role-based-authorization-sample](https://stackblitz.com/edit/angular-8-role-based-authorization-sample))

## How to run the app locally

1. Install NodeJS and NPM from [https://nodejs.org](https://nodejs.org).
2. Download or clone the tutorial project source code from [https://github.com/FSou1/angular-8-role-based-authorization-sample](https://github.com/FSou1/angular-8-role-based-authorization-sample)
3. Install all required npm packages by running `npm install` from the command line in the project root folder (where the package.json is located).
4. Start the application by running `npm start` from the command line in the project root folder, this will build the application and automatically launch it in the browser on the URL http://localhost:4200.

NOTE: You can also run the app directly using the Angular CLI command `ng serve --open`. To do this first install the Angular CLI globally on your system with the command `npm install -g @angular/cli`.

> Thanks to Jason Watmore for the guide above

## Project structure

<a name="projectstructure"></a>

Below are the main project files that contain the application logic:

- src
  - app
    - admin
      - dashboard
        - dashboard.component.html
        - dashboard.component.ts
      - [admin-routing.module.ts](#admin-routing-module)
      - [admin.module.ts](#admin-module)
    - app
      - [app.component.html](#app-component-template)
      - [app.component.ts](#app-component)
    - directives
      - [user-role.directive.ts](#user-role-directive)
      - [user.directive.ts](#user-directive)
    - error
      - not-found
        - not-found.component.html
        - not-found.component.ts
    - home
      - home.component.html
      - home.component.ts
    - login
      - [login.component.html](#login-component-template)
      - [login.component.ts](#login-component)
    - models
      - [role.ts](#role-model)
      - [user.ts](#user-model)
    - profile
      - [profile.component.html](#profile-template)
      - [profile.component.ts](#profile-component)
    - services
      - [auth.service.ts](#authservice)
    - [app-routing.guard.ts](#auth-guard)
    - [app-routing.module.ts](#app-routing-module)
    - [app.module.ts](#app-module)
  - index.html
- package.json
- tsconfig.json

<a name="admin-routing-module"></a>

### Admin routing module

The module contains admin routes and mapped components. The module itself is [lazy loaded](https://angular.io/guide/lazy-loading-ngmodules) and managed as a part of the [AppRoutingModule](#app-routing-module) (by passing the [AuthGuard](#auth-guard) to the `canActivate` and the `canLoad` properties).

```typescript
import { Routes } from "@angular/router";
import { DashboardComponent } from "./dashboard/dashboard.component";

export const routes: Routes = [
  { path: "dashboard", component: DashboardComponent },
];
```

[Back to top](#projectstructure)

<a name="admin-module"></a>

### Admin module

The admin module is the root module for the admin workspace and declares the list of available components and routes.

```typescript
import { NgModule } from "@angular/core";
import { DashboardComponent } from "./dashboard/dashboard.component";
import { RouterModule } from "@angular/router";
import { routes } from "./admin-routing.module";

@NgModule({
  declarations: [DashboardComponent],
  imports: [RouterModule.forChild(routes)],
  providers: [],
})
export class AdminModule {}
```

[Back to top](#projectstructure)

<a name="app-component-template"></a>

### App component template

The app component template is the root component template of the application. It contains the main nav bar which changes based on the user.

The profile and the logout links are only visible for authorized users (by using the `isAuthorized` getter). The dashboard link is only displayed for admins (by using the `isAdmin` getter). The login link is only accessible by anonymous users.

The router-outlet directive acts as a placeholder that the app (Angular) dynamically fills based on the current URL (router state). [Read more](https://angular.io/guide/router).

```html
<header class="navbar navbar-expand navbar-dark bg-dark">
  <ul class="navbar-nav mr-auto">
    <li class="nav-item">
      <a class="nav-link" [routerLink]="['/']">Home</a>
    </li>

    <!-- Authorized only -->
    <li class="nav-item" *ngIf="isAuthorized">
      <a class="nav-link" [routerLink]="['profile']">Profile</a>
    </li>

    <!-- Admin only -->
    <li class="nav-item" *ngIf="isAdmin">
      <a class="nav-link" [routerLink]="['admin', 'dashboard']">Dashboard</a>
    </li>

    <!-- Anonymous only -->
    <li class="nav-item" *ngIf="!isAuthorized">
      <a class="nav-link" [routerLink]="['login']">Login</a>
    </li>

    <!-- Authorized only -->
    <li class="nav-item" *ngIf="isAuthorized">
      <a class="nav-link" (click)="logout()">Logout</a>
    </li>
  </ul>

  <span class="navbar-text">
    <span class="badge badge-warning" *ngIf="isAuthorized">Authorized</span>
    <span class="badge badge-danger" *ngIf="isAdmin">Admin</span>
  </span>
</header>

<div class="jumbotron">
  <div class="container-fluid">
    <router-outlet></router-outlet>
  </div>
</div>
```

[Back to top](#projectstructure)

<a name="app-component"></a>

### App component

The app component is the root component of the application. It contains the `isAuthorized` and the `isAdmin` getters which are called inside the template. The `logout` method is used to sign out the currently authorized user and redirect him to the login page.

```typescript
import { Component, OnInit } from "@angular/core";
import { Router } from "@angular/router";
import { Role } from "../models/role";
import { AuthService } from "../services/auth.service";

@Component({
  selector: "app-root",
  templateUrl: "./app.component.html",
  styleUrls: ["./app.component.css"],
})
export class AppComponent {
  constructor(private router: Router, private authService: AuthService) {}

  get isAuthorized() {
    return this.authService.isAuthorized();
  }

  get isAdmin() {
    return this.authService.hasRole(Role.Admin);
  }

  logout() {
    this.authService.logout();
    this.router.navigate(["login"]);
  }
}
```

[Back to top](#projectstructure)

<a name="user-role-directive"></a>

### User role directive

The user role directive is a structural directive that conditionally includes a template if:

- A user is authorized
- A user has at least one of the allowed roles

Conditions are evaluated by using the [AuthService](#authservice).

Allowed roles are passed by using the `appUserRole` setter, for example: `*appUserRole="[ Role.Admin ]"`.

A good thing is that the directive encapsulates the authorization logic and can be reused within the application.

On the other hand, a `hasAccess` value is evaluated within the `ngOnInit` method, so a template won't be redrawn if the value changes. That's why I wouldn't use the directive in the layout (like the navbar). At the same time, it's still useful on the [profile component template](#profile-template).

```typescript
import { Directive, OnInit, Input } from "@angular/core";
import { TemplateRef, ViewContainerRef } from "@angular/core";
import { AuthService } from "../services/auth.service";
import { Role } from "../models/role";

@Directive({ selector: "[appUserRole]" })
export class UserRoleDirective implements OnInit {
  constructor(
    private templateRef: TemplateRef<any>,
    private authService: AuthService,
    private viewContainer: ViewContainerRef
  ) {}

  userRoles: Role[];

  @Input()
  set appUserRole(roles: Role[]) {
    if (!roles || !roles.length) {
      throw new Error("Roles value is empty or missed");
    }

    this.userRoles = roles;
  }

  ngOnInit() {
    let hasAccess = false;

    if (this.authService.isAuthorized() && this.userRoles) {
      hasAccess = this.userRoles.some((r) => this.authService.hasRole(r));
    }

    if (hasAccess) {
      this.viewContainer.createEmbeddedView(this.templateRef);
    } else {
      this.viewContainer.clear();
    }
  }
}
```

[Back to top](#projectstructure)

<a name="user-directive"></a>

### User directive

The user directive is like the [UserRoleDirective](#user-role-directive). It includes a template for authorized users only. You can see the `*appUser` directive in action at the [profile component template](#profile-template).

```typescript
import { Directive, OnInit, Input } from "@angular/core";
import { TemplateRef, ViewContainerRef } from "@angular/core";
import { AuthService } from "../services/auth.service";

@Directive({ selector: "[appUser]" })
export class UserDirective implements OnInit {
  constructor(
    private templateRef: TemplateRef<any>,
    private authService: AuthService,
    private viewContainer: ViewContainerRef
  ) {}

  ngOnInit() {
    const hasAccess = this.authService.isAuthorized();

    if (hasAccess) {
      this.viewContainer.createEmbeddedView(this.templateRef);
    } else {
      this.viewContainer.clear();
    }
  }
}
```

[Back to top](#projectstructure)

<a name="login-component-template"></a>

### Login component template

The login component template contains two buttons: login as a user or as an admin. It sets a user and redirects to the '/' URL when a button is clicked.

```html
<div class="alert alert-success" role="alert">Login component!</div>

<button class="btn btn-outline-warning" (click)="login(Role.User)">
  Login as User
</button>

<button class="btn btn-outline-danger" (click)="login(Role.Admin)">
  Login as Admin
</button>
```

[Back to top](#projectstructure)

<a name="login-component"></a>

### Login component

The login component uses the [AuthService](#authservice) to set a user with a specific role. Then the user is navigated to the home page by using the [@angular/router](https://angular.io/api/router).

```typescript
import { Component, OnInit } from "@angular/core";
import { Router } from "@angular/router";
import { Role } from "../models/role";
import { AuthService } from "../services/auth.service";

@Component({
  selector: "app-login",
  templateUrl: "./login.component.html",
  styleUrls: ["./login.component.css"],
})
export class LoginComponent {
  Role = Role;

  constructor(private router: Router, private authService: AuthService) {}

  login(role: Role) {
    this.authService.login(role);
    this.router.navigate(["/"]);
  }
}
```

[Back to top](#projectstructure)

<a name="role-model"></a>

### Role model

The role model contains an enum that defines the roles that are supported by the application.

```typescript
export enum Role {
  User = 1,
  Admin = 2,
}
```

[Back to top](#projectstructure)

<a name="user-model"></a>

### User model

The user model contains a user class declaration. The `role` property is the only thing required for the demo needs.

```typescript
import { Role } from "./role";

export class User {
  role: Role;
}
```

[Back to top](#projectstructure)

<a name="profile-template"></a>

### Profile component template

The profile component template uses the `*appUser` and the `*appUserRole` structural directives to control access to specific blocks.

```html
<div class="alert alert-success" role="alert">Profile component!</div>

<div class="alert alert-warning" role="alert" *appUser>
  Content for authorized users only (e.g. display email, send message)!
</div>

<div class="alert alert-danger" role="alert" *appUserRole="[ Role.Admin ]">
  Content for admin users only (e.g. enable/disable user)!
</div>
```

[Back to top](#projectstructure)

<a name="profile-component"></a>

### Profile component

The profile component may not contain a single property.

NOTE: The property `Role` is used within the template and helps to avoid [magic strings](https://en.wikipedia.org/wiki/Magic_string). Otherwise, the admin value should be presented as a string (`*appUserRole="[ 'Admin' ]"`) or a number (`*appUserRole="[ 1 ]"`).

```typescript
import { Component, OnInit } from "@angular/core";
import { Role } from "../models/role";

@Component({
  selector: "app-profile",
  templateUrl: "./profile.component.html",
  styleUrls: ["./profile.component.css"],
})
export class ProfileComponent {
  Role = Role;
}
```

[Back to top](#projectstructure)

<a name="authservice"></a>

### Auth service

The authentication service is used to manage the user information (the private `user` property).

It's specifically made as simple as possible to address the demo needs:

- The `login(role: Role)` method sets a user with a specific role
- The `isAuthorized()` and the `hasRole(role: Role)` methods encapsulate the authorization logic
- The `logout()` method resets a user

```typescript
import { Injectable } from "@angular/core";
import { User } from "../models/user";
import { Role } from "../models/role";

@Injectable()
export class AuthService {
  private user: User;

  isAuthorized() {
    return !!this.user;
  }

  hasRole(role: Role) {
    return this.isAuthorized() && this.user.Role === role;
  }

  login(role: Role) {
    this.user = { role: role };
  }

  logout() {
    this.user = null;
  }
}
```

[Back to top](#projectstructure)

<a name="auth-guard"></a>

### Auth guard

The auth guard is an angular route guard that's used to restrict users from accessing secured routes or modules.

It's done by implementing the `canActivate` method (and the `CanActivate` interface) which decides if a specific route can be activated by the current user.

The `canLoad` method (and the `CanLoad` interface) is used to decide whether to load components of a specific module.

The auth guard uses the [AuthService](#authservice) to ensure the current user is authorized. Otherwise, access is denied (by returning `false`) and the user is redirected to the login page.

To check if a user has a specific role we can use the `data` property of the requested route (e.g. `route.data.roles as Role[]`).

As you could see further the auth guard is used in the [AppRoutingModule](#app-routing-module) to protect the profile page and the dashboard page (which is part of the [AdminModule](#admin-module)).

```typescript
import { CanActivate, CanLoad } from "@angular/router";
import { Router, ActivatedRouteSnapshot, Route } from "@angular/router";
import { Observable } from "rxjs";
import { Injectable } from "@angular/core";
import { AuthService } from "./services/auth.service";
import { Role } from "./models/role";

@Injectable()
export class AuthGuard implements CanActivate, CanLoad {
  constructor(private router: Router, private authService: AuthService) {}

  canActivate(
    route: ActivatedRouteSnapshot
  ): Observable<boolean> | Promise<boolean> | boolean {
    if (!this.authService.isAuthorized()) {
      this.router.navigate(["login"]);
      return false;
    }

    const roles = route.data.roles as Role[];
    if (roles && !roles.some((r) => this.authService.hasRole(r))) {
      this.router.navigate(["error", "not-found"]);
      return false;
    }

    return true;
  }

  canLoad(route: Route): Observable<boolean> | Promise<boolean> | boolean {
    if (!this.authService.isAuthorized()) {
      return false;
    }

    const roles = route.data && (route.data.roles as Role[]);
    if (roles && !roles.some((r) => this.authService.hasRole(r))) {
      return false;
    }

    return true;
  }
}
```

[Back to top](#projectstructure)

<a name="app-routing-module"></a>

### App routing module

The module contains root routes and mapped components so the Angular Router knows which component to display based on the current URL.

The home and login pages are available for all users. The profile page is secured by the [AuthGuard](#auth-guard) and is available for authorized users only. The admin module routes and components are lazy loaded and available for admin users only.

For more information on Angular Routing and Navigation see [https://angular.io/guide/router](https://angular.io/guide/router).

```typescript
import { NgModule } from "@angular/core";
import { Routes, RouterModule } from "@angular/router";
import { HomeComponent } from "./home/home.component";
import { ProfileComponent } from "./profile/profile.component";
import { NotFoundComponent } from "./error/not-found/not-found.component";
import { AuthGuard } from "./app-routing.guard";
import { AuthService } from "./services/auth.service";
import { LoginComponent } from "./login/login.component";
import { Role } from "./models/role";

const routes: Routes = [
  {
    path: "",
    children: [
      {
        path: "",
        component: HomeComponent,
      },

      {
        path: "profile",
        canActivate: [AuthGuard],
        component: ProfileComponent,
      },

      {
        path: "login",
        component: LoginComponent,
      },
    ],
  },
  {
    path: "admin",
    canLoad: [AuthGuard],
    canActivate: [AuthGuard],
    data: {
      roles: [Role.Admin],
    },
    loadChildren: () =>
      import("./admin/admin.module").then((m) => m.AdminModule),
  },
  {
    path: "**",
    component: NotFoundComponent,
  },
];

@NgModule({
  imports: [RouterModule.forRoot(routes)],
  exports: [RouterModule],
  providers: [AuthGuard, AuthService],
})
export class AppRoutingModule {}
```

[Back to top](#projectstructure)

<a name="app-module"></a>

### App module

The app module is the root module for the application and declares the list of available components and routes.

```typescript
import { BrowserModule } from "@angular/platform-browser";
import { NgModule } from "@angular/core";
import { AppRoutingModule } from "./app-routing.module";
import { AppComponent } from "./app/app.component";
import { HomeComponent } from "./home/home.component";
import { ProfileComponent } from "./profile/profile.component";
import { NotFoundComponent } from "./error/not-found/not-found.component";
import { LoginComponent } from "./login/login.component";
import { UserRoleDirective } from "./directives/user-role.directive";
import { UserDirective } from "./directives/user.directive";
import { AuthService } from "./services/auth.service";

@NgModule({
  declarations: [
    AppComponent,
    HomeComponent,
    ProfileComponent,
    NotFoundComponent,
    LoginComponent,
    UserDirective,
    UserRoleDirective,
  ],
  imports: [BrowserModule, AppRoutingModule],
  exports: [UserDirective, UserRoleDirective],
  providers: [AuthService],
  bootstrap: [AppComponent],
})
export class AppModule {}
```

[Back to top](#projectstructure)

## Comments

Discuss this post on [Hacker News](https://news.ycombinator.com/item?id=22410458), [/r/programming](https://www.reddit.com/r/programming/comments/f930ok/angular_8_rolebased_authorization_tutorial/), [/r/angular](https://www.reddit.com/r/angular/comments/f936pd/angular_8_rolebased_authorization_tutorial/) or [/r/frontend](https://www.reddit.com/r/Frontend/comments/f93700/angular_8_rolebased_authorization_tutorial/)

## Reference:

- [Stackblitz - sample](https://stackblitz.com/edit/angular-8-role-based-authorization-sample)
- [GitHub - source code](https://github.com/FSou1/angular-8-role-based-authorization-sample)
- [Jason Watmore's Blog - Angular 8 - RBA Tutorial with Example](https://jasonwatmore.com/post/2019/08/06/angular-8-role-based-authorization-tutorial-with-example)
