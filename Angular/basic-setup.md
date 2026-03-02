# Project Setup & Core Architecture

- Concept 1: Angular Project StructureWhat it is & why it exists:
Angular CLI generates a structured project. In modern Angular (17+), we use standalone components — no NgModule required. You organize features into folders, and each feature has its own components, services, and routes.For your Student MS, the structure will look like

```bash

src/app/
├── core/                  # Singleton services, interceptors, guards
│   ├── interceptors/
│   ├── guards/
│   └── services/          # Auth service, etc.
├── features/              # Feature modules (lazy loaded)
│   ├── auth/
│   ├── students/
│   ├── teachers/
│   ├── courses/
│   └── enrollments/
├── shared/                # Reusable components, pipes, models
│   ├── components/
│   ├── models/            # TypeScript interfaces matching your DTOs
│   └── pipes/
├── app.component.ts
├── app.config.ts          # Root app config (replaces AppModule)
└── app.routes.ts          # Root route definitions

```

- app.config.ts — The New AppModule What it is:
In Angular 17+, AppModule is replaced by app.config.ts, which is a plain ApplicationConfig object. You register providers here — router, HTTP client, etc.

- Concept 3: Lazy Loading Routes What it is & why:
Instead of loading all feature code upfront, lazy loading means Angular only loads a feature's code when the user navigates to it. This keeps the initial bundle small. Each feature exports its own routes array, and the root router references it with loadChildren.

## Project scaffolding

We typically use the angular cli:

```bash

npm i -g @angular/cli

```

Node and npm are a prerq.
After the above command complete's, we can make an angular app like:

```bash

ng new <app_name> # use '.' for creating the app inside the root directory.

```

Then cd into the app and run **ng serve**. The development server will open at port 4200.
For faster scaffolding:

```bash

ng new my-app --routing --style=css --skip-tests

```

This skips the initial QnA prompts.

## Generate Components Later

Instead of manually creating files:

```bash
ng generate component navbar

# or shorthand

ng g c navbar

```
