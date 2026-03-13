# Angular Form Handling

## Table of Contents

- [Overview](#overview)
- [Setup](#setup)
- [Template-Driven Forms](#template-driven-forms)
- [Reactive Forms](#reactive-forms)
- [Built-In Validators](#built-in-validators)
- [Custom Validators](#custom-validators)
- [Cross-Field Validation](#cross-field-validation)
- [Dynamic Forms With FormArray](#dynamic-forms-with-formarray)
- [Form Updates and State](#form-updates-and-state)
- [Handling Submission](#handling-submission)
- [Observing Form Changes](#observing-form-changes)
- [Custom Form Controls (ControlValueAccessor)](#custom-form-controls-controlvalueaccessor)
- [Template-Driven vs Reactive](#template-driven-vs-reactive)
- [Best Practices](#best-practices)
- [Common Pitfalls](#common-pitfalls)

---

## Overview

Angular supports two main approaches for form handling:

1. Template-driven forms
2. Reactive forms

Both approaches support validation, submission, and error display.

- Template-driven forms are simpler for small forms and rely heavily on template directives.
- Reactive forms are code-first, easier to test, and better for large/dynamic forms.

---

## Setup

### Standalone Component Setup

Import form modules directly in the component.

```typescript
import { Component } from "@angular/core";
import { FormsModule, ReactiveFormsModule } from "@angular/forms";

@Component({
  selector: "app-example",
  standalone: true,
  imports: [FormsModule, ReactiveFormsModule],
  template: `...`,
})
export class ExampleComponent {}
```

### NgModule Setup (legacy/module-based projects)

```typescript
import { NgModule } from "@angular/core";
import { BrowserModule } from "@angular/platform-browser";
import { FormsModule, ReactiveFormsModule } from "@angular/forms";

@NgModule({
  declarations: [AppComponent],
  imports: [BrowserModule, FormsModule, ReactiveFormsModule],
  bootstrap: [AppComponent],
})
export class AppModule {}
```

---

## Template-Driven Forms

Template-driven forms use `ngModel`, `name`, and `ngForm`.

### Example: Simple Registration Form

```html
<form #registerForm="ngForm" (ngSubmit)="onSubmit(registerForm)">
  <label>
    Name
    <input
      type="text"
      name="name"
      required
      minlength="3"
      [(ngModel)]="model.name"
      #name="ngModel"
    />
  </label>

  <div *ngIf="name.invalid && (name.dirty || name.touched)">
    <small *ngIf="name.errors?.['required']">Name is required.</small>
    <small *ngIf="name.errors?.['minlength']">Minimum 3 characters.</small>
  </div>

  <label>
    Email
    <input
      type="email"
      name="email"
      required
      email
      [(ngModel)]="model.email"
      #email="ngModel"
    />
  </label>

  <div *ngIf="email.invalid && (email.dirty || email.touched)">
    <small *ngIf="email.errors?.['required']">Email is required.</small>
    <small *ngIf="email.errors?.['email']">Enter a valid email.</small>
  </div>

  <button type="submit" [disabled]="registerForm.invalid">Register</button>
</form>
```

```typescript
import { Component } from "@angular/core";
import { FormsModule, NgForm } from "@angular/forms";

@Component({
  selector: "app-register-template",
  standalone: true,
  imports: [FormsModule],
  templateUrl: "./register-template.component.html",
})
export class RegisterTemplateComponent {
  model = {
    name: "",
    email: "",
  };

  onSubmit(form: NgForm): void {
    if (form.invalid) {
      return;
    }

    console.log("Template form submit:", this.model);
    form.resetForm();
  }
}
```

---

## Reactive Forms

Reactive forms define structure and rules in TypeScript using `FormGroup`, `FormControl`, and `FormBuilder`.

### Example: Login Form

```typescript
import { Component } from "@angular/core";
import { FormBuilder, ReactiveFormsModule, Validators } from "@angular/forms";

@Component({
  selector: "app-login-reactive",
  standalone: true,
  imports: [ReactiveFormsModule],
  template: `
    <form [formGroup]="loginForm" (ngSubmit)="onSubmit()">
      <label>
        Email
        <input type="email" formControlName="email" />
      </label>

      <div *ngIf="email.invalid && (email.dirty || email.touched)">
        <small *ngIf="email.errors?.['required']">Email is required.</small>
        <small *ngIf="email.errors?.['email']">Invalid email format.</small>
      </div>

      <label>
        Password
        <input type="password" formControlName="password" />
      </label>

      <div *ngIf="password.invalid && (password.dirty || password.touched)">
        <small *ngIf="password.errors?.['required']"
          >Password is required.</small
        >
        <small *ngIf="password.errors?.['minlength']"
          >Minimum 6 characters.</small
        >
      </div>

      <button type="submit" [disabled]="loginForm.invalid">Login</button>
    </form>
  `,
})
export class LoginReactiveComponent {
  constructor(private fb: FormBuilder) {}

  loginForm = this.fb.group({
    email: ["", [Validators.required, Validators.email]],
    password: ["", [Validators.required, Validators.minLength(6)]],
  });

  get email() {
    return this.loginForm.controls.email;
  }

  get password() {
    return this.loginForm.controls.password;
  }

  onSubmit(): void {
    if (this.loginForm.invalid) {
      this.loginForm.markAllAsTouched();
      return;
    }

    console.log("Reactive form submit:", this.loginForm.value);
    this.loginForm.reset();
  }
}
```

---

## Built-In Validators

Angular provides common validators via `Validators`:

- `Validators.required`
- `Validators.requiredTrue`
- `Validators.email`
- `Validators.min(number)`
- `Validators.max(number)`
- `Validators.minLength(number)`
- `Validators.maxLength(number)`
- `Validators.pattern(regexOrString)`
- `Validators.nullValidator`

Example:

```typescript
const form = this.fb.group({
  username: ["", [Validators.required, Validators.minLength(4)]],
  age: [18, [Validators.min(18), Validators.max(99)]],
});
```

---

## Custom Validators

### Synchronous Custom Validator

```typescript
import { AbstractControl, ValidationErrors, ValidatorFn } from "@angular/forms";

export function noWhitespaceValidator(): ValidatorFn {
  return (control: AbstractControl): ValidationErrors | null => {
    const value = (control.value || "") as string;
    const isOnlyWhitespace = value.trim().length === 0;
    return isOnlyWhitespace ? { whitespace: true } : null;
  };
}
```

Usage:

```typescript
name: ["", [Validators.required, noWhitespaceValidator()]];
```

### Async Custom Validator

Useful when validation depends on a server check, like username uniqueness.

```typescript
import {
  AbstractControl,
  AsyncValidatorFn,
  ValidationErrors,
} from "@angular/forms";
import { Observable, of } from "rxjs";
import { delay, map } from "rxjs/operators";

export function usernameTakenValidator(taken: string[]): AsyncValidatorFn {
  return (control: AbstractControl): Observable<ValidationErrors | null> => {
    return of(control.value).pipe(
      delay(300),
      map((value: string) =>
        taken.includes((value || "").toLowerCase())
          ? { usernameTaken: true }
          : null,
      ),
    );
  };
}
```

Usage:

```typescript
username: this.fb.control(
  "",
  [Validators.required],
  [usernameTakenValidator(["admin", "root"])],
);
```

---

## Cross-Field Validation

Cross-field validators are attached to the `FormGroup` when one field depends on another.

Example: password and confirmPassword must match.

```typescript
import { AbstractControl, ValidationErrors, ValidatorFn } from "@angular/forms";

export const passwordMatchValidator: ValidatorFn = (
  group: AbstractControl,
): ValidationErrors | null => {
  const password = group.get("password")?.value;
  const confirmPassword = group.get("confirmPassword")?.value;

  if (!password || !confirmPassword) {
    return null;
  }

  return password === confirmPassword ? null : { passwordMismatch: true };
};
```

Usage:

```typescript
registerForm = this.fb.group(
  {
    password: ["", [Validators.required, Validators.minLength(8)]],
    confirmPassword: ["", [Validators.required]],
  },
  { validators: passwordMatchValidator },
);
```

Template error:

```html
<div *ngIf="registerForm.errors?.['passwordMismatch'] && registerForm.touched">
  Password and confirm password do not match.
</div>
```

---

## Dynamic Forms With FormArray

Use `FormArray` when users can add/remove repeated fields dynamically.

Example: multiple phone numbers.

```typescript
import { Component } from "@angular/core";
import {
  FormArray,
  FormBuilder,
  ReactiveFormsModule,
  Validators,
} from "@angular/forms";

@Component({
  selector: "app-phones",
  standalone: true,
  imports: [ReactiveFormsModule],
  template: `
    <form [formGroup]="form" (ngSubmit)="save()">
      <div formArrayName="phones">
        <div *ngFor="let ctrl of phones.controls; let i = index">
          <input [formControlName]="i" placeholder="Phone number" />
          <button type="button" (click)="removePhone(i)">Remove</button>
        </div>
      </div>

      <button type="button" (click)="addPhone()">Add phone</button>
      <button type="submit">Save</button>
    </form>
  `,
})
export class PhonesComponent {
  constructor(private fb: FormBuilder) {}

  form = this.fb.group({
    phones: this.fb.array([this.fb.control("", [Validators.required])]),
  });

  get phones(): FormArray {
    return this.form.get("phones") as FormArray;
  }

  addPhone(): void {
    this.phones.push(this.fb.control("", [Validators.required]));
  }

  removePhone(index: number): void {
    this.phones.removeAt(index);
  }

  save(): void {
    if (this.form.invalid) {
      this.form.markAllAsTouched();
      return;
    }

    console.log(this.form.value);
  }
}
```

---

## Form Updates and State

### `setValue` vs `patchValue`

- `setValue` requires all controls to be provided.
- `patchValue` updates only provided controls.

```typescript
this.profileForm.setValue({
  firstName: "John",
  lastName: "Doe",
  email: "john@example.com",
});

this.profileForm.patchValue({
  email: "new@example.com",
});
```

### Disable/Enable Controls

```typescript
this.profileForm.get("email")?.disable();
this.profileForm.get("email")?.enable();
```

### Control State Flags

Useful flags:

- `valid` / `invalid`
- `pristine` / `dirty`
- `untouched` / `touched`
- `pending` (async validators running)
- `disabled` / `enabled`

### `updateOn` Strategy

Control when validators run:

- `'change'` (default)
- `'blur'`
- `'submit'`

```typescript
email: [
  "",
  {
    validators: [Validators.required, Validators.email],
    updateOn: "blur",
  },
];
```

---

## Handling Submission

A common robust pattern:

```typescript
submit(): void {
  if (this.form.invalid) {
    this.form.markAllAsTouched();
    return;
  }

  const payload = this.form.getRawValue(); // includes disabled controls
  this.api.save(payload).subscribe({
    next: () => this.form.reset(),
    error: (err) => console.error('Save failed', err),
  });
}
```

Tips:

- Use `markAllAsTouched()` so validation messages appear on submit.
- Use `getRawValue()` if you need disabled control values too.
- Disable submit button while request is in progress to prevent duplicates.

---

## Observing Form Changes

Use `valueChanges` and `statusChanges` to react to user input.

```typescript
import { debounceTime, distinctUntilChanged } from "rxjs/operators";

this.searchForm
  .get("query")
  ?.valueChanges.pipe(debounceTime(300), distinctUntilChanged())
  .subscribe((query) => {
    console.log("Search query:", query);
  });

this.form.statusChanges.subscribe((status) => {
  console.log("Form status:", status); // VALID | INVALID | PENDING | DISABLED
});
```

Common use cases:

- Debounced API search
- Live previews
- Conditional UI behavior

---

## Custom Form Controls (ControlValueAccessor)

If you build reusable inputs (date picker, rating, toggle), implement `ControlValueAccessor` so Angular forms can read/write value and touched state.

```typescript
import { Component, forwardRef } from "@angular/core";
import { ControlValueAccessor, NG_VALUE_ACCESSOR } from "@angular/forms";

@Component({
  selector: "app-rating-input",
  standalone: true,
  template: `
    <button type="button" (click)="setRating(1)">1</button>
    <button type="button" (click)="setRating(2)">2</button>
    <button type="button" (click)="setRating(3)">3</button>
    <button type="button" (click)="setRating(4)">4</button>
    <button type="button" (click)="setRating(5)">5</button>
  `,
  providers: [
    {
      provide: NG_VALUE_ACCESSOR,
      useExisting: forwardRef(() => RatingInputComponent),
      multi: true,
    },
  ],
})
export class RatingInputComponent implements ControlValueAccessor {
  value = 0;
  disabled = false;

  private onChange: (value: number) => void = () => {};
  private onTouched: () => void = () => {};

  writeValue(value: number): void {
    this.value = value ?? 0;
  }

  registerOnChange(fn: (value: number) => void): void {
    this.onChange = fn;
  }

  registerOnTouched(fn: () => void): void {
    this.onTouched = fn;
  }

  setDisabledState(isDisabled: boolean): void {
    this.disabled = isDisabled;
  }

  setRating(rating: number): void {
    if (this.disabled) {
      return;
    }

    this.value = rating;
    this.onChange(this.value);
    this.onTouched();
  }
}
```

Then use it like any normal control:

```html
<form [formGroup]="form">
  <app-rating-input formControlName="rating"></app-rating-input>
</form>
```

---

## Template-Driven vs Reactive

| Feature                        | Template-Driven | Reactive         |
| ------------------------------ | --------------- | ---------------- |
| Form model location            | HTML template   | TypeScript class |
| Setup complexity               | Low             | Medium           |
| Scalability                    | Moderate        | High             |
| Dynamic forms                  | Harder          | Easier           |
| Unit testing                   | Harder          | Easier           |
| Control over validation timing | Limited         | Strong           |

Use template-driven for simple forms and reactive for complex or enterprise-level forms.

---

## Best Practices

1. Prefer reactive forms for large business forms.
2. Keep validation logic close to form definitions.
3. Show errors only after user interaction (`touched`/`dirty`) or submit.
4. Use `FormArray` for repeated dynamic fields.
5. Debounce expensive operations tied to `valueChanges`.
6. Keep form DTO types aligned with backend contracts.
7. Split very large forms into child components.

---

## Common Pitfalls

1. Missing `name` attribute in template-driven inputs.
2. Mixing `[(ngModel)]` and `formControlName` on the same control.
3. Forgetting to import `FormsModule` or `ReactiveFormsModule`.
4. Showing validation errors before user interaction.
5. Using `setValue` with incomplete objects.
6. Not unsubscribing from long-lived custom subscriptions.

---

This file gives you a complete foundation for handling forms in Angular, from basic inputs to advanced validation and reusable custom controls.
